WITH params AS (
  SELECT to_date(replace(:DateRef, ' 00:00:00',''), 'DD/MM/YYYY') AS date_ref
),
-- Adresse la plus récente par garant
latest_addr AS (
  SELECT actid, adrid
  FROM (
    SELECT actid, adrid,
           ROW_NUMBER() OVER (PARTITION BY actid ORDER BY aadordre DESC) AS rn
    FROM actadresse
    WHERE aadflagsiege = 1
      AND aaddtremplace IS NULL
  ) x
  WHERE rn = 1
),
unpaid AS (
  SELECT fre.fredosid AS dosid,
         COALESCE(SUM(F_PIFactImp(fac.facid, p.date_ref)), 0) AS montant_impayes_ttc
  FROM facreference fre
  JOIN facture fac ON fre.facid = fac.facid
    AND fac.facflagavoir IS NULL
    AND fac.facidorigine IS NULL
  JOIN role r ON fac.rolcode = r.rolcode
    AND r.rolcodeexterne IN ('CLIENT','GARANT')
  CROSS JOIN params p
  GROUP BY fre.fredosid
),
indemn AS (
  SELECT fre.fredosid AS dosid,
         COALESCE(SUM(F_PlFactImp(fac.facid, p.date_ref)), 0) AS indemnite
  FROM facreference fre
  JOIN facture fac ON fre.facid = fac.facid
    AND fac.facflagavoir IS NULL
    AND fac.facidorigine IS NULL
  JOIN facligne fl ON fl.facid = fac.facid
  JOIN rubrique rub ON fl.rubid = rub.rubid
    AND rub.rubcode = 'INDRES'
  JOIN acteur a ON a.actid = fac.actidgestion
  JOIN role r ON fac.rolcode = r.rolcode
    AND r.rolcodeexterne IN ('CLIENT','GARANT')
  CROSS JOIN params p
  WHERE a.ugecode = :UniteGestion
  GROUP BY fre.fredosid
),
base AS (
  SELECT
    lta.taclibelle               AS libelle_activite_dos,
    lpg.tpglibelle               AS libelle_profil_de_gestion_dos,
    soc.actnom                   AS libelle_soc,
    dossier.dosnum || '/' || LPAD(TO_CHAR(dossier.dosavenant),2,'0') AS reference_dos,
    dossier.dosnom               AS nom_dos,
    dossier.dosdtfin             AS date_fin_dos,
    dossier.dossectgestion       AS sect_gest,
    dp2.jallibelle               AS libelle_jalon_dosp,
    ph2.phalibelle               AS libelle_phase_dosp,
    ds.dphdteffet                AS date_effet_phase_dosp,
    dossier.dospool              AS pooldos,
    g.actcode                    AS code_gar,
    g.actnom                     AS nom_gar,
    g.actsiret                   AS siret_gar,
    dag.dagordre                 AS ordre_grntie_act_dos_acdg,
    tg.tgalibelle                AS libelle_garantie_acdg,
    phg.phalibelle               AS lib_phase_grntie_acdg,
    dag.dagdtdeb                 AS date_debut_grntie_acdg,
    dag.dagdtfin                 AS date_fin_grntie_acdg,
    dag.dagpctgarantieutilise    AS pourcentage_grntie_utilise,
    dag.dagmTassiette            AS mntnt_assiette_grntie_acdg,
    CASE
      WHEN dag.dagdtfin < add_months(p.date_ref, -6) THEN 'echu plus de 6 mois'
      WHEN dag.dagdtfin BETWEEN add_months(p.date_ref, -6) AND p.date_ref - 1 THEN 'echu moins de 6 mois'
      ELSE 'active'
    END                          AS etat_garantie_acdg,
    add_months(p.date_ref, -1)   AS en_plus,
    adr.advoie                   AS voie,
    adr.adrcodepostal            AS code_postal,
    adr.adville                  AS ville,
    dossier.actid                AS actiddossier,
    dossier.taccode              AS taccode,
    dossier.devcode              AS devcode,
    dossier.dosid                AS dosid
  FROM dossier
  JOIN params p ON 1=1
  JOIN lantactivite lta ON lta.taccode = dossier.taccode AND lta.lancode = 'FR'
  JOIN lantprofilgestion lpg ON lpg.tpgcode = dossier.tpgcode AND lpg.lancode = 'FR'
  JOIN acteur soc ON soc.actid = dossier.actid AND soc.actnom <> 'GENEFIMMO'
  JOIN dosphase ds ON ds.dosid = dossier.dosid
  JOIN lanjalon dp2 ON dp2.jalcode = ds.jalcode AND dp2.lancode = 'FR'
  JOIN lanphase ph2 ON ph2.phacode = ds.phacode AND ph2.phadest = ds.phadest AND ph2.lancode = 'FR'
  JOIN dosactacteur dac ON dac.dosid = dossier.dosid
    AND dac.rolcode IN ('AGENCE','CAUTION','TIEASSU','GARANT','AGGARHG')
    AND dac.dactdeb < p.date_ref
    AND (dac.dacdtfin IS NULL OR dac.dacdtfin > p.date_ref)
  JOIN acteur g ON g.actid = dac.actid
  JOIN dosactgarantie dag ON dag.dosid = dac.dosid AND dag.dacordre = dac.dacordre
  JOIN lantgarantie tg ON tg.tgacode = dag.tacode AND tg.lancode = 'FR'
  JOIN lanphase phg ON phg.phacode = dag.phacode AND phg.lancode = 'FR'
  LEFT JOIN latest_addr la ON la.actid = g.actid
  LEFT JOIN adresse adr ON adr.adrid = la.adrid
  WHERE ds.dphdteffet <= p.date_ref
    AND (ds.dphdtfin IS NULL OR ds.dphdtfin >= p.date_ref)
),
final AS (
  SELECT
    b.libelle_activite_dos,
    b.libelle_profil_de_gestion_dos,
    b.libelle_soc,
    b.reference_dos,
    b.nom_dos,
    b.date_fin_dos,
    b.sect_gest,
    b.libelle_jalon_dosp,
    b.libelle_phase_dosp,
    b.date_effet_phase_dosp,
    b.pooldos,
    b.code_gar,
    b.nom_gar,
    b.siret_gar,
    b.ordre_grntie_act_dos_acdg,
    b.libelle_garantie_acdg,
    b.lib_phase_grntie_acdg,
    b.date_debut_grntie_acdg,
    b.date_fin_grntie_acdg,
    b.mntnt_assiette_grntie_acdg,
    b.pourcentage_grntie_utilise,
    COALESCE(u.montant_impayes_ttc, 0)                  AS montant_impayes_ttc,
    ROUND(COALESCE(u.montant_impayes_ttc, 0), 2)       AS montant_impayes_ttc_rounded,
    ROUND(COALESCE(u.montant_impayes_ttc, 0), 2) - COALESCE(i.indemnite, 0) AS impayes_ttc,
    dev.F_PLECF_EG148(b.dosid, b.actiddossier, b.devcode, b.devcode, b.taccode, p.date_ref) AS encours_brut,
    b.mntnt_assiette_grntie_acdg * (1 - COALESCE(b.pourcentage_grntie_utilise, 0) / 100) AS montant_de_garantie,
    b.etat_garantie_acdg,
    b.en_plus,
    b.voie,
    b.code_postal,
    b.ville
  FROM base b
  LEFT JOIN unpaid u ON u.dosid = b.dosid
  LEFT JOIN indemn i ON i.dosid = b.dosid
  JOIN params p ON 1=1
)
SELECT *
FROM final
ORDER BY libelle_soc,
         reference_dos,
         nom_dos,
         libelle_jalon_dosp,
         code_gar,
         nom_gar,
         siret_gar,
         ordre_grntie_act_dos_acdg,
         libelle_garantie_acdg;
