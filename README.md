SELECT DISTINCT
  lta.taclibelle                                     AS libelle_activite_dos,
  lpg.tpglibelle                                     AS libelle_profil_de_gestion_dos,
  soc.actnom                                         AS libelle_soc,
  dossier.dosnum || '/' || LPAD(TO_CHAR(dossier.dosavenant),2,'0') AS reference_dos,
  dossier.dosnom                                     AS nom_dos,
  dossier.dosdtfin                                   AS date_fin_dos,
  dossier.dossectgestion                             AS sect_gest,
  dp.jallibelle                                      AS libelle_jalon_dosp,
  ph.phalibelle                                      AS libelle_phase_dosp,
  ds.dphdteffet                                      AS date_effet_phase_dosp,
  dossier.dospool                                    AS pooldos,
  g.actcode                                          AS code_gar,
  g.actnom                                           AS nom_gar,
  g.actsiret                                         AS siret_gar,
  dag.dagordre                                       AS ordre_grntie_act_dos_acdg,
  tg.tgalibelle                                      AS libelle_garantie_acdg,
  phg.phalibelle                                     AS lib_phase_grntie_acdg,
  dag.dagdtdeb                                       AS date_debut_grntie_acdg,
  dag.dagdtfin                                       AS date_fin_grntie_acdg,
  dag.dagpctgarantieutilise                          AS pourcentage_grntie_utilise,
  dag.dagmTassiette                                  AS mntnt_assiette_grntie_acdg,
  CASE
    WHEN dag.dagdtfin < ADD_MONTHS(TO_DATE(REPLACE(:DateRef,' 00:00:00',''),'DD/MM/YYYY'),-6) THEN 'echu plus de 6 mois'
    WHEN dag.dagdtfin BETWEEN ADD_MONTHS(TO_DATE(REPLACE(:DateRef,' 00:00:00',''),'DD/MM/YYYY'),-6)
         AND TO_DATE(REPLACE(:DateRef,' 00:00:00',''),'DD/MM/YYYY')-1 THEN 'echu moins de 6 mois'
    ELSE 'active'
  END                                                 AS etat_garantie_acdg,
  ADD_MONTHS(TO_DATE(REPLACE(:DateRef,' 00:00:00',''),'DD/MM/YYYY'),-1) AS en_plus,
  adr.advoie                                         AS voie,
  adr.adrcodepostal                                  AS code_postal,
  adr.adville                                        AS ville,
  COALESCE(indemn.indemnite,0)                       AS indemnite,
  ROUND(COALESCE(unpaid.montant_impayes_ttc,0),2)     AS montant_impayes_ttc,
  ROUND(COALESCE(unpaid.montant_impayes_ttc,0),2)
    - COALESCE(indemn.indemnite,0)                    AS impayes_ttc,
  dev.F_PLECF_EG148(
    dossier.dosid,
    dossier.actid,
    dossier.devcode,
    dossier.devcode,
    dossier.taccode,
    TO_DATE(REPLACE(:DateRef,' 00:00:00',''),'DD/MM/YYYY')
  )                                                   AS encours_brut,
  dag.dagmTassiette * (1 - NVL(dag.dagpctgarantieutilise,0)/100) AS montant_de_garantie
FROM dossier
JOIN utilisateur_dossier ud
  ON ud.ugecode = dossier.ugecode
 AND ud.bologin = :Utilisateur
JOIN acteur soc
  ON soc.actid = dossier.actid
 AND soc.actnom <> 'GENEFIMMO'
JOIN lantactivite lta
  ON lta.taccode = dossier.taccode
 AND lta.lancode = 'FR'
JOIN lantprofilgestion lpg
  ON lpg.tpgcode = dossier.tpgcode
 AND lpg.lancode = 'FR'
JOIN dosphase ds
  ON ds.dosid = dossier.dosid
 AND ds.dphdteffet <= TO_DATE(REPLACE(:DateRef,' 00:00:00',''),'DD/MM/YYYY')
 AND (ds.dphdtfin IS NULL OR ds.dphdtfin >= TO_DATE(REPLACE(:DateRef,' 00:00:00',''),'DD/MM/YYYY'))
JOIN lanjalon dp
  ON dp.jalcode = ds.jalcode
 AND dp.lancode = 'FR'
JOIN lanphase ph
  ON ph.phacode = ds.phacode
 AND ph.phadest = ds.phadest
 AND ph.lancode = 'FR'
JOIN dosactacteur dac
  ON dac.dosid = dossier.dosid
 AND dac.rolcode IN ('AGENCE','CAUTION','TIEASSU','GARANT','AGGARHG')
 AND dac.dactdeb < TO_DATE(REPLACE(:DateRef,' 00:00:00',''),'DD/MM/YYYY')
 AND (dac.dacdtfin IS NULL OR dac.dacdtfin > TO_DATE(REPLACE(:DateRef,' 00:00:00',''),'DD/MM/YYYY'))
JOIN acteur g
  ON g.actid = dac.actid
JOIN dosactgarantie dag
  ON dag.dosid = dac.dosid
 AND dag.dacordre = dac.dacordre
JOIN lantgarantie tg
  ON tg.tgacode = dag.tacode
 AND tg.lancode = 'FR'
JOIN lanphase phg
  ON phg.phacode = dag.phacode
 AND phg.lancode = 'FR'
LEFT JOIN (
  SELECT actid, adrid
  FROM (
    SELECT actid,
           adrid,
           ROW_NUMBER() OVER (PARTITION BY actid ORDER BY aadordre DESC) AS rn
    FROM actadresse
    WHERE aadflagsiege = 1
      AND aaddtremplace IS NULL
  ) x
  WHERE rn = 1
) latest_addr
  ON latest_addr.actid = g.actid
LEFT JOIN adresse adr
  ON adr.adrid = latest_addr.adrid
LEFT JOIN (
  SELECT fre.fredosid AS dosid,
         SUM(
           F_PIFactImp(
             fac.facid,
             TO_DATE(REPLACE(:DateRef,' 00:00:00',''),'DD/MM/YYYY')
           )
         ) AS montant_impayes_ttc
  FROM facreference fre
  JOIN facture fac
    ON fre.facid = fac.facid
   AND fac.facflagavoir IS NULL
   AND fac.facidorigine IS NULL
  JOIN role r
    ON fac.rolcode = r.rolcode
   AND r.rolcodeexterne IN ('CLIENT','GARANT')
  GROUP BY fre.fredosid
) unpaid
  ON unpaid.dosid = dossier.dosid
LEFT JOIN (
  SELECT fre.fredosid AS dosid,
         SUM(
           F_PlFactImp(
             fac.facid,
             TO_DATE(REPLACE(:DateRef,' 00:00:00',''),'DD/MM/YYYY')
           )
         ) AS indemnite
  FROM facreference fre
  JOIN facture fac
    ON fre.facid = fac.facid
   AND fac.facflagavoir IS NULL
   AND fac.facidorigine IS NULL
  JOIN facligne fl
    ON fl.facid = fac.facid
  JOIN rubrique rub
    ON fl.rubid = rub.rubid
   AND rub.rubcode = 'INDRES'
  JOIN acteur a2
    ON a2.actid = fac.actidgestion
  JOIN role r2
    ON fac.rolcode = r2.rolcode
   AND r2.rolcodeexterne IN ('CLIENT','GARANT')
  WHERE a2.ugecode = :UniteGestion
  GROUP BY fre.fredosid
) indemn
  ON indemn.dosid = dossier.dosid
ORDER BY
  soc.actnom,
  reference_dos,
  dossier.dosnom,
  dp.jallibelle,
  g.actcode,
  g.actnom,
  g.actsiret,
  dag.dagordre,
  tg.tgalibelle;
