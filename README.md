SELECT DISTINCT
    la.taclibelle                               AS libelle_activite_dos,
    lpg.tpglibelle                              AS libelle_profil_de_gestion_dos,
    soc.actnom                                  AS libelle_soc,
    d.dosnum || '/' || LPAD(TO_CHAR(d.dosavenant), 2, '0') AS reference_dos,
    d.dosnom                                    AS nom_dos,
    d.dosdtfin                                  AS date_fin_dos,
    d.dossectgestion                            AS sect_gest,
    CASE d.taccode
      WHEN 'CBI'  THEN 'CLIENT'
      WHEN 'PRET' THEN 'EMPRUNT'
      ELSE NULL
    END                                          AS rolcode,
    da_client.actid                             AS actid,
    d.actid                                     AS actiddossier,
    d.taccode,
    d.devcode,
    lj.jallibelle                               AS libelle_jalon_dosp,
    lp.phalibelle                               AS libelle_phase_dosp,
    dp.dphdteffet                               AS date_effet_phase_dosp,
    d.dospool                                   AS pooldos,
    gar.actcode                                 AS code_gar,
    gar.actnom                                  AS nom_gar,
    gar.actsiret                                AS siret_gar,
    dag.dagordre                                AS ordre_grntie_act_dos_acdg,
    tg.tgalibelle                               AS libelle_garantie_acdg,
    tgph.phalibelle                             AS lib_phase_grntie_acdg,
    dag.dagdtdeb                                AS date_debut_grntie_acdg,
    dag.dagdtfin                                AS date_fin_grntie_acdg,
    dag.dagpctgarantieutilise                   AS pourcentage_grntie_utilise,
    dag.dagmtassiette                           AS mntnt_assiette_grntie_acdg,
    NVL(imp.impayes_ttc, 0)                     AS montant_impayes_ttc,
    CASE
      WHEN dag.dagdtfin
           < ADD_MONTHS(
               TO_DATE(REPLACE($P{DateRef},' 00:00:00',''),
                       'DD/MM/YYYY'),
               -6
             )
        THEN 'echu plus de 6 mois'
      WHEN dag.dagdtfin
           BETWEEN
             ADD_MONTHS(
               TO_DATE(REPLACE($P{DateRef},' 00:00:00',''),
                       'DD/MM/YYYY'),
               -6
             )
             AND
             TO_DATE(REPLACE($P{DateRef},' 00:00:00',''),
                     'DD/MM/YYYY')
             - INTERVAL '1' DAY
        THEN 'echu moins de 6 mois'
      ELSE 'active'
    END                                          AS etat_garantie_acdg,
    TO_DATE(REPLACE($P{DateRef},' 00:00:00',''),'DD/MM/YYYY')
      - INTERVAL '1' DAY                          AS en_plus,
    adr.advoie                                   AS voie,
    adr.adrcodepost                              AS code_postal,
    adr.adville                                  AS ville,
    NVL(ind.indemnite, 0)                        AS indemnite,
    ROUND(NVL(imp.impayes_ttc, 0),2)
      - NVL(ind.indemnite, 0)                     AS impayes_ttc,
    dev.F_PLECF_EG148(
      d.dosid,
      d.actiddossier,
      d.devcode,
      d.devcode,
      d.taccode,
      TO_DATE(REPLACE($P{DateRef},' 00:00:00',''),'DD/MM/YYYY')
    )                                            AS encours_brut
FROM DOSSIER d
JOIN UTILISATEUR ud
  ON ud.ugecode = d.ugecode
 AND ud.bologin = $P{Utilisateur}
 AND ud.ugecode = $P{UniteGestion}
JOIN ACTEUR soc
  ON soc.actid = d.actid
 AND soc.actnom <> 'GENEFIMMO'
JOIN LANTACTIVITE la
  ON la.taccode = d.taccode
 AND la.lancode = 'FR'
JOIN LANTPROFILGESTION lpg
  ON lpg.tpgcode = d.tpgcode
 AND lpg.lancode = 'FR'
-- Phase active au moment de DateRef
JOIN DOSPHASE dp
  ON dp.dosid = d.dosid
 AND dp.jalcode NOT IN ('ARCEC','ARC')
 AND dp.dphdteffet <= TO_DATE(REPLACE($P{DateRef},' 00:00:00',''),'DD/MM/YYYY')
 AND (dp.dphdtfin IS NULL
       OR dp.dphdtfin >= TO_DATE(REPLACE($P{DateRef},' 00:00:00',''),'DD/MM/YYYY'))
JOIN LANJALON lj
  ON lj.jalcode = dp.jalcode
 AND lj.lancode = 'FR'
JOIN LANPHASE lp
  ON lp.phacode = dp.phacode
 AND lp.lancode = 'FR'
-- Acteur « client » courant
JOIN DOSACTEUR da_client
  ON da_client.dosid = d.dosid
 AND da_client.rolcode =
     CASE d.taccode WHEN 'CBI' THEN 'CLIENT' WHEN 'PRET' THEN 'EMPRUNT' END
 AND da_client.dactfin IS NULL
-- Acteur garant courant
JOIN DOSACTEUR da_garant
  ON da_garant.dosid = d.dosid
 AND da_garant.rolcode IN ('AGENCE','CAUTION','TIEASSU','GARANT','AGGARHG')
JOIN ACTEUR gar
  ON gar.actid = da_garant.actid
LEFT JOIN ACTADRESSE aad
  ON aad.actid = gar.actid
 AND aad.aadflagsiege = 1
 AND aad.aaddtremplace IS NULL
LEFT JOIN ADRESSE adr
  ON adr.adrid = aad.adrid
-- Garantie active ou échu
JOIN DOSACTGARANTIE dag
  ON dag.dosid = da_garant.dosid
 AND dag.dacordre = da_garant.dacordre
 AND dag.dagdtdeb < TO_DATE(REPLACE($P{DateRef},' 00:00:00',''),'DD/MM/YYYY')
 AND (dag.dagdtfin IS NULL
       OR dag.dagdtfin > TO_DATE(REPLACE($P{DateRef},' 00:00:00',''),'DD/MM/YYYY'))
LEFT JOIN LANTGARANTIE tg
  ON tg.tgacode = dag.tacode
 AND tg.lancode = 'FR'
LEFT JOIN LANPHASE tgph
  ON tgph.phacode = dag.phacode
 AND tgph.lancode = 'FR'
-- Montant impayés TTC
LEFT JOIN (
  SELECT fre.fredosid    AS dosid,
         SUM(F_PlFact_Imp(f.facid,
               TO_DATE(REPLACE($P{DateRef},' 00:00:00',''),
                       'DD/MM/YYYY')
         ))            AS impayes_ttc
  FROM FACREFERENCE fre
  JOIN FACTURE f
    ON f.facid = fre.facid
  JOIN ROLE r
    ON r.rolcode = f.rolcode
   AND r.rolcodeexterne IN ('CLIENT','GARANT')
  GROUP BY fre.fredosid
) imp
  ON imp.dosid = d.dosid
-- Indemnité (RUBRIQUE = 'INDRES')
LEFT JOIN (
  SELECT fre.fredosid    AS dosid,
         SUM(F_PlFact_Imp(f.facid,
               TO_DATE(REPLACE($P{DateRef},' 00:00:00',''),
                       'DD/MM/YYYY')
         ))            AS indemnite
  FROM FACREFERENCE fre
  JOIN FACTURE f
    ON f.facid       = fre.facid
   AND f.facidorigine IS NULL
   AND f.facflagavoir IS NULL
  JOIN FACLIGNE fl
    ON fl.facid = f.facid
  JOIN RUBRIQUE rub
    ON rub.rubid   = fl.rubid
   AND rub.ugecode = $P{UniteGestion}
   AND rub.rubcode = 'INDRES'
  JOIN ROLE r2
    ON r2.rolcode = f.rolcode
   AND r2.rolcodeexterne IN ('CLIENT','GARANT')
  GROUP BY fre.fredosid
) ind
  ON ind.dosid = d.dosid
WHERE
  (
    (
      dag.dagdtfin
        < ADD_MONTHS(
            TO_DATE(REPLACE($P{DateRef},' 00:00:00',''),
                    'DD/MM/YYYY'
            ),
            -6
          )
      OR dag.dagdtfin
         BETWEEN
           ADD_MONTHS(
             TO_DATE(REPLACE($P{DateRef},' 00:00:00',''),
                     'DD/MM/YYYY'
             ),
             -6
           )
           AND
           TO_DATE(REPLACE($P{DateRef},' 00:00:00',''),
                   'DD/MM/YYYY'
           )
           - INTERVAL '1' DAY
    )
    AND NOT EXISTS (
      SELECT 1
      FROM DOSACTGARANTIE dag2
      WHERE dag2.dosid = d.dosid
        AND TO_DATE(REPLACE($P{DateRef},' 00:00:00',''),
                     'DD/MM/YYYY'
            ) BETWEEN dag2.dagdtdeb
                      AND NVL(dag2.dagdtfin,
                              TO_DATE('31122500','DDMMYYYY')
                            )
    )
  )
  OR NOT (
    dag.dagdtfin
      < ADD_MONTHS(
          TO_DATE(REPLACE($P{DateRef},' 00:00:00',''),
                  'DD/MM/YYYY'
          ),
          -6
        )
    OR dag.dagdtfin
       BETWEEN
         ADD_MONTHS(
           TO_DATE(REPLACE($P{DateRef},' 00:00:00',''),
                   'DD/MM/YYYY'
           ),
           -6
         )
         AND
         TO_DATE(REPLACE($P{DateRef},' 00:00:00',''),
                 'DD/MM/YYYY'
         )
         - INTERVAL '1' DAY
  )
ORDER BY
    soc.actnom,
    reference_dos,
    nom_dos,
    libelle_jalon_dosp,
    code_gar,
    nom_gar,
    siret_gar,
    ordre_grntie_act_dos_acdg,
    libelle_garantie_acdg;
