DEFINE dateRefClean = TO_DATE(REPLACE($P{DateRef}, ' 00:00:00', ''), 'DD/MM/YYYY');

SELECT DISTINCT
    a.libelle_activite_dos,
    a.libelle_profil_de_gestion_dos,
    a.libelle_soc,
    a.reference_dos,
    a.nom_DOS,
    a.date_fin_dos,
    a.sect_gest,
    a.libelle_jalon_dosp,
    a.libelle_phase_dosp,
    a.date_effet_phase_dosp,
    a.pooldos,
    a.code_gar,
    a.nom_gar,
    a.siret_gar,
    a.ordre_grntie_act_dos_acdg,
    a.libelle_garantie_acdg,
    a.lib_phase_grntie_acdg,
    a.date_debut_grntie_acdg,
    a.date_fin_grntie_acdg,
    a.mntnt_assiette_grntie_acdg,
    a.pourcentage_grntie_utilise,
    a.etat_garantie_acdg,
    a.en_plus,
    a.mntnt_assiette_grntie_acdg * (1 - NVL(a.pourcentage_grntie_utilise, 0) / 100) AS montant_de_garantie,
    a.voie,
    a.code_postal,
    a.ville,
    a.indemnite,
    ROUND(a.montant_impayes_ttc, 2) AS montant_impayes_ttc,
    ROUND(a.montant_impayes_ttc, 2) - a.indemnite AS impayes_ttc,
    dev.F_PLECF_EG148(a.DOSID, a.actiddossier, a.DEVCODE, a.DEVCODE, a.TACCODE, &dateRefClean) AS encours_brut
FROM (
    SELECT
        LANTACTIVITE.TACLIBELLE AS libelle_activite_dos,
        LANTPROFILGESTION.TPGLIBELLE AS libelle_profil_de_gestion_dos,
        SOCIETE.ACTNOM AS libelle_soc,
        DOSSIER.DOSID,
        DOSSIER.DOSNUM || '/' || LPAD(TO_CHAR(DOSSIER.DOSAVENANT), 2, '0') AS reference_dos,
        DOSSIER.DOSNOM AS nom_DOS,
        DOSSIER.DOSDTFIN AS date_fin_dos,
        DOSSIER.DOSSECTGESTION AS sect_gest,
        LANJALON.JALLIBELLE AS libelle_jalon_dosp,
        LANPHASE.PHALIBELLE AS libelle_phase_dosp,
        DOSPHASE.DPHDTEFFET AS date_effet_phase_dosp,
        DOSSIER.DOSPOOL AS pooldos,
        GARANT.ACTCODE AS code_gar,
        GARANT.ACTNOM AS nom_gar,
        GARANT.ACTSIRET AS siret_gar,
        DOSACTGARANTIE.DAGORDRE AS ordre_grntie_act_dos_acdg,
        LANTGARANTIE.TGALIBELLE AS libelle_garantie_acdg,
        LANPHASE_GARANTIE.PHALIBELLE AS lib_phase_grntie_acdg,
        DOSACTGARANTIE.DAGDTDEB AS date_debut_grntie_acdg,
        DOSACTGARANTIE.DAGDTFIN AS date_fin_grntie_acdg,
        DOSACTGARANTIE.DAGMTASSIETTE AS mntnt_assiette_grntie_acdg,
        DOSACTGARANTIE.DAGPCTGARANTIEUTILISE AS pourcentage_grntie_utilise,
        CASE
            WHEN DOSACTGARANTIE.DAGDTFIN < ADD_MONTHS(&dateRefClean, -6) THEN 'echu plus de 6 mois'
            WHEN DOSACTGARANTIE.DAGDTFIN BETWEEN ADD_MONTHS(&dateRefClean, -6) AND (&dateRefClean - 1) THEN 'echu moins de 6 mois'
            ELSE 'active'
        END AS etat_garantie_acdg,
        ADD_MONTHS(&dateRefClean, -1) AS en_plus,
        ADVOIE AS voie,
        ADRCODEPOST AS code_postal,
        ADVILLE AS ville,
        IMP.montant_impayes_ttc,
        IND.indemnite,
        DOSSIER.ACTID AS actiddossier,
        DOSSIER.TACCODE,
        DOSSIER.DEVCODE
    FROM DOSSIER
    JOIN LANTACTIVITE ON LANTACTIVITE.TACCODE = DOSSIER.TACCODE
    JOIN LANTPROFILGESTION ON LANTPROFILGESTION.TPGCODE = DOSSIER.TPGCODE
    JOIN SOCIETE ON SOCIETE.ACTID = DOSSIER.ACTID
    JOIN ACTEURGESTION ON ACTEURGESTION.ACTID = SOCIETE.ACTID
    JOIN UTILISATEUR UTILISATEUR_DOSSIER ON UTILISATEUR_DOSSIER.UGECODE = SOCIETE.UGECODE AND UTILISATEUR_DOSSIER.BOLOGIN = $P{Utilisateur}
    JOIN DOSPHASE ON DOSPHASE.DOSID = DOSSIER.DOSID
    JOIN LANPHASE ON LANPHASE.PHACODE = DOSPHASE.PHACODE AND LANPHASE.PHADEST = DOSPHASE.PHADEST
    JOIN LANJALON ON LANJALON.JALCODE = DOSPHASE.JALCODE
    LEFT JOIN DOSACTEUR ON DOSACTEUR.DOSID = DOSSIER.DOSID
    LEFT JOIN ACTEUR GARANT ON GARANT.ACTID = DOSACTEUR.ACTID
    LEFT JOIN DOSACTGARANTIE ON DOSACTGARANTIE.DOSID = DOSACTEUR.DOSID AND DOSACTGARANTIE.DACORDRE = DOSACTEUR.DACORDRE
    LEFT JOIN LANTGARANTIE ON LANTGARANTIE.TGACODE = DOSACTGARANTIE.TACODE
    LEFT JOIN LANPHASE LANPHASE_GARANTIE ON LANPHASE_GARANTIE.PHACODE = DOSACTGARANTIE.PHACODE AND LANPHASE_GARANTIE.PHADEST = 'DOSSIER'
    LEFT JOIN ACTADRESSE ON ACTADRESSE.ACTID = GARANT.ACTID AND ACTADRESSE.AADFLAGSIEGE = 1 AND ACTADRESSE.AADDTREMPLACE IS NULL
    LEFT JOIN ADRESSE ON ADRESSE.ADRID = ACTADRESSE.ADRID
    LEFT JOIN (
        SELECT FRE.FREDOSID, SUM(NVL(F_PIFactImp(FAC.FACID, $P{DateRef}), 0)) AS montant_impayes_ttc
        FROM FACREFERENCE FRE
        JOIN FACTURE FAC ON FRE.FACID = FAC.FACID
        JOIN ROLE R ON R.ROLCODE = FAC.ROLCODE
        WHERE R.ROLCODEEXTERNE IN ('CLIENT', 'GARANT')
        GROUP BY FRE.FREDOSID
    ) IMP ON IMP.FREDOSID = DOSSIER.DOSID
    LEFT JOIN (
        SELECT FRE.FREDOSID, SUM(NVL(F_PIFactImp(FAC.FACID, $P{DateRef}), 0)) AS indemnite
        FROM FACREFERENCE FRE
        JOIN FACTURE FAC ON FRE.FACID = FAC.FACID
        JOIN FACLIGNE FL ON FL.FACID = FAC.FACID
        JOIN RUBRIQUE RUB ON RUB.RUBID = FL.RUBID
        JOIN ACTEUR A ON A.ACTID = FAC.ACTIDGESTION
        JOIN ROLE R ON R.ROLCODE = FAC.ROLCODE
        WHERE RUB.UGECODE = $P{UniteGestion}
          AND FAC.FACIDORIGINE IS NULL
          AND FAC.FACFLAGAVOIR IS NULL
          AND R.ROLCODEEXTERNE IN ('CLIENT', 'GARANT')
          AND RUB.RUBCODE = 'INDRES'
        GROUP BY FRE.FREDOSID
    ) IND ON IND.FREDOSID = DOSSIER.DOSID
) a
ORDER BY
    a.libelle_soc,
    a.reference_dos,
    a.nom_DOS,
    a.libelle_jalon_dosp,
    a.code_gar,
    a.nom_gar,
    a.siret_gar,
    a.ordre_grntie_act_dos_acdg,
    a.libelle_garantie_acdg;
