# Notes de passation vers Mario — arbitrages et points de vigilance

**Document de travail. Regroupe les décisions et contournements pris pendant la migration UAT (Wario) à transférer dans la documentation finale de passation. Chaque entrée = un point que Mario doit connaître pour la migration production.**

Rappel du cadre : Wario livre une migration **UAT** fin août. Mario refait la vraie migration en **production** (~janvier), après ses propres tests. Toutes les décisions ci-dessous sont réversibles et à revalider en prod.

---

## 1. Validation rules contournées par bypass (Opportunité)

**Décision** : le bypass de validation rules (`Bypass_validation_rules__c` sur le user d'intégration) est activé pour la migration des opportunités, avec l'accord de l'équipe Pluxee.

**Pourquoi** : plusieurs règles exigent des champs qui n'ont **aucune source dans Glady** (champs `PLUXEE ONLY` liés à la restructuration produit en cours) — impossible à remplir depuis la source.

**Ce que Mario doit savoir** :
- Le bypass **masque** les champs manquants : les oppos s'insèrent mais certains champs restent vides sans alerte. Le `resultats_ko.csv` ne signale plus ces trous.
- À faire en prod : réactiver les règles et retester, une fois la restructuration produit Pluxee terminée et les champs cibles alimentés.
- **À compléter** : liste précise des règles bypassées + champs laissés vides (en cours de constitution par Wario).

**Exemple de règle concernée** : validation sur Opportunity exigeant `Type_de_produit__c`, `BV_estime__c`, `EmetteurenPlace__c`, `Tarifmetteurenplace__c`, `Support__c`, `Dotation_par_benficiaire__c`, `Nombre_de_beneficiaires__c`, `Datede1erecommandeestimee__c`… quand StageName avancé + Famille = Gift Incentive. Champs `PLUXEE ONLY` : `EmetteurenPlace__c`, `Tarifmetteurenplace__c`, `Dotation_par_benficiaire__c`, `Equip__c`.

---

## 2. RecordTypeId non mappé sur les Leads

**Décision (constatée a posteriori)** : les leads ont été migrés **sans mapper `RecordTypeId`** → ils ont tous pris le record type par défaut de l'org cible.

**Ce que Mario doit savoir** :
- Si le record type porte une logique métier (page layouts, picklists restreintes par RT), c'est à corriger en prod.
- Peut expliquer certaines validation rules qui se déclenchaient sur les leads.
- Sur l'Opportunité, l'erreur a été corrigée : `RecordTypeId` est résolu par **lookup sur `DeveloperName`** (clé stable entre orgs, contrairement à l'Id technique).

**Recommandation prod** : mapper `RecordTypeId` sur tous les objets via lookup DeveloperName.

---

## 3. Cascades de picklists dépendantes (Famille → Produit → Marché)

**Décision** : gérées via PicklistMapper avec la colonne `target_field` (une valeur source alimente plusieurs champs cibles). Les valeurs `STIMI`/`STIME` de `Marche__c` ne sont insérables que si `Produit__c = CCI` et `Famille_de_produit__c = Gift Incentive` — les 3 champs sont renseignés ensemble depuis la seule valeur source `Marche__c`.

**Ce que Mario doit savoir** :
- Ces lignes du CSV ne sont **pas** de la traduction de valeur, mais des règles structurelles.
- La restructuration produit Pluxee (prévue prod fin juillet / mi-août) peut **invalider ces mappings**. À revoir après la refonte.
- Les lignes d'une cascade sont indépendantes dans le fichier → toujours les modifier ensemble.

---

## 4. Champs mis de côté (non migrés en UAT)

| Champ | Raison |
|---|---|
| `TypeLead__c`, `Fonctions__c` (Lead) | absents côté cible, liés à la restructuration produit |
| Famille/Produit/Marché | restructuration Pluxee en cours |
| `Rating`, `Pronouns`, `GenderIdentity` (Lead) | aucune donnée d'un côté ou de l'autre |
| `Concurrents__c`, `InteretProduits__c` (Lead) | 0 donnée source |
| Champs `PLUXEE ONLY` (Oppo) | aucune source Glady |

**Recommandation prod** : reprendre ces champs une fois la restructuration Pluxee terminée.

---

## 5. Code NAF (Lead)

**Décision** : `Code_NAF__c` côté Pluxee est un **lookup** vers un référentiel d'objets `Code_NAF__c` (pas un champ texte). Le code Glady (`Code_APE__c`, format avec point `10.92Z`) est normalisé (point retiré) et résolu par lookup vers l'Id du référentiel.

**Ce que Mario doit savoir** :
- Référentiel de 738 enregistrements chargé dans MigrWario. 100 % de résolution quand le code source existe.
- Prérequis prod : le référentiel `Code_NAF__c` doit être peuplé dans l'org cible avant migration.
- Validation rule `NAFIsMandatoryOnConversion` : les leads Converti sans code APE côté Glady sont rejetés → donnée source à nettoyer (rapport métier).

---

## 6. Filtres de périmètre appliqués

| Objet | Filtre | Décision |
|---|---|---|
| Lead | `RecordType.Name != 'Enseigne'` | métier — périmètre sales |
| Lead | `LeadSource != 'Migration Sodexo'` | métier — 3 leads exclus |
| Opportunity | `NOT RecordType.Name LIKE '%Sodexo%' AND NOT LIKE '%Enseigne%'` | métier |
| Task/Event | `Who.Type = 'Lead'` + `IsRecurrence = false` | récurrences exclues (contexte utilisateur différent, plus de sens) |

**⚠️ Filtres de TEST à ne PAS garder en prod** : `AND Marche__c != null`, `AND Email != ''` — ajoutés ponctuellement pour débloquer des tests, ils excluent des données. À retirer pour le run complet.

---

## 7. Activités (Task/Event)

**Décisions** :
- Périmètre Lead uniquement (activités liées aux leads). Celles des Compte/Contact/Oppo traitées dans un job séparé.
- `WhatId` toujours vide sur les leads → non mappé.
- `OwnerId`/`CreatedById` = user de déploiement. `CreatedDate` conservée (permission "Set Audit Fields upon Record Creation" active).
- Récurrences (`IsRecurrence = true`) exclues.

**Constat métier utile** : les leads qui ont des activités sont massivement des leads **sans email** (leads travaillés/sourcés). Un filtre sur email élimine la population qui a des tâches.

---

## 8. Champs d'audit / traçabilité

**Décision** : chaque objet migré porte un champ `IdGlady__c` (ou `ID_Salesforce_Glady__c`) contenant l'Id 18 caractères de la source, pour le rejeu et les lookups en cascade (Task→Lead, Devis→Oppo, Oppo→Account).

**Ce que Mario doit savoir** :
- Sur Lead, le fichier `resultats_ok.csv` avait un souci (colonne id_source vide) → contourné par une requête directe sur MigrWario (`SELECT Id, IdGlady__c FROM Lead WHERE IdGlady__c != null`).
- `ID_18__c` côté Glady est une **formule** (`CASESAFEID(Id)`) : requêtable mais non insérable. Sert de clé source, pas de champ cible.

---

## 9. Rejets qualité de données → hors périmètre Wario

**Décision** : les rejets dus à des données source incomplètes (code APE vide, date de clôture manquante, email/téléphone absents, doublons, longueurs) ne sont **pas** corrigés par Wario.

**Ce que Mario doit savoir** :
- Un rapport qualité de données est produit pour le métier (cleaning source côté Glady).
- Mario prévoit de re-cleaner les données en prod (scripts Python : doublons, longueurs). L'environnement de test doit être rafraîchi avec ces données nettoyées (~mi-août).

---

## Points encore ouverts (à trancher avec Guillaume, retour 10 août)

- Validation rule « raison de non conversion » (Lead) : accepter les rejets UAT ou valeur de repli ?
- `Type de produit` / `Type de business` (Oppo) : API names à confirmer.
- `Support__c` (Oppo) : lequel des 4 champs source `Support_*`, ou cascade de priorité ?
- Multipicklists Oppo (`Concurrents__c`, `Interet_Produits__c`, `EvenementsURSSAFCSE__c`) : besoin d'un `translateMulti` si volume.


## 10. Multipicklists Opportunité — extraction première valeur

**Décision** : les champs multipicklist source dont la cible est une picklist simple sont migrés en ne conservant que la **première valeur** (split sur `;` dans le tMap, avant PicklistMapper).

**Cas concerné** :
- `Concurrents__c` (multipicklist, 18 valeurs) -> `EmetteurRetenu__c` (picklist simple)

**Implémentation tMap** (pas de modif PicklistMapper) :
```java
PicklistMapper.translate("Opportunity","Concurrents__c",
   row1.Concurrents__c == null ? null :
   (row1.Concurrents__c.contains(";") ? row1.Concurrents__c.split(";")[0] : row1.Concurrents__c))
```

**Ce que Mario doit savoir** :
- Perte d'information assumée : une oppo avec plusieurs concurrents n'en conserve qu'un.
- La "première" valeur = ordre de stockage Salesforce (ordre de définition de la picklist), PAS l'ordre de sélection ni une priorité métier. Rien ne garantit que ce soit le concurrent principal.
- À valider avec le métier : veulent-ils cette valeur, une autre, ou vide plutôt qu'une valeur arbitraire ?
- Volume réel d'oppos multi-concurrents à vérifier (si la plupart n'ont qu'une valeur, l'impact est négligeable).

**Cas NON traité par ce contournement** :
- `Interet_Produits__c` (multipicklist) -> `Produits__c` (multipicklist **dépendante du champ `Context__c`**). Double difficulté (multi + dépendance). Mis de côté.
- `EvenementsURSSAFCSE__c` (multipicklist) -> `Evenement__c` (multipicklist) : à traiter si volume, nécessiterait un vrai translateMulti.


## 11. Picklists Opportunité — décisions par champ (GROUP BY réel)

Toutes les décisions ci-dessous s'appuient sur le GROUP BY des données réelles, pas sur le fichier de mapping métier (jugé non fiable).

| Champ source | Cible | Décision |
|---|---|---|
| `Marche__c` (PAS `Marche2__c`) | `Marche__c` | Mappé avec cascade (Interne->STIMI, Externe->STIME + Produit__c=CCI + Famille=Gift Incentive). Identique au Lead. |
| `Marche2__c` | - | **Vide dans les données -> non mappé.** Le fichier métier disait `Marche2__c->Marche__c`, c'était faux : c'est `Marche__c` qui porte les données. |
| `Loss_Reason__c` | - | **Vide (79947 oppos) -> non mappé.** |
| `OrigineLead__c` | `Origine__c` | **Non mappé (option laisser vide).** 98,5% vide (78735/80000), 5 valeurs résiduelles (Phone/Website/Mail/Event/Mapping) sans correspondance cible évidente. Risque > gain. |
| `Source__c` | `Origine__c` | **Mappé (30 lignes).** Valeurs quasi identiques à Origine__C du Lead -> réutilisation de la colonne `Origine_du_lead__c` du Lead comme base (1 source -> 1 cible ici, pas 3). |

### Détail des arbitrages Source__c -> Origine__c

- **21 valeurs** reprises directement du mapping Lead (cible Origine_du_lead__c) : Sourcing, SEO, Scraping, Scraping + AS, Salon, SEA, Inbound, BDD + AS, Other, Referral, Mapping, Affiliation, SVI (Aircall), Email, Prospect gagné, AO, Communication, SMA, PRM, Slack, LinkedIn.
- **7 valeurs déduites** (non mappées côté Lead, correspondance raisonnable) :
  - BDD Pipe (8276), BDD Growth (3716), Growth (232), BDD (6) -> `Import Marketing` (logique des autres BDD)
  - Outbound (2611) -> `Téléprospection` (prospection sortante)
  - Acquisition (16), Google (9) -> `Digital`
- **1 pari** : Sales (156) -> `Action co` (action commerciale) — à confirmer métier, sinon défaut vide.
- **Défaut** : `XX_DEFAULT_XX -> vide` (couvre les ~8443 oppos vides + valeurs futures).

**Ce que Mario doit savoir** : le mapping Source__c réutilise la logique Lead. Si le métier valide/corrige l'Origine du Lead, répercuter sur Source__c Oppo. Le pari `Sales -> Action co` est à confirmer.


## 12. Concurrents__c -> EmetteurRetenu__c (multipicklist -> picklist simple)

**Décision** : split première valeur (voir arbitrage #10), correspondances validées avec le métier (tableau fourni), corrigées pour ne pas créer de nouvelles valeurs côté Pluxee.

**Impact du split confirmé** : sur 126 combinaisons distinctes rencontrées, **108 sont multi-valeurs (85%)**. La perte n'est donc PAS marginale : une majorité des oppos avec concurrent en ont plusieurs, un seul est conservé. La "première" valeur = ordre de stockage Salesforce (ordre de définition picklist), pas une priorité métier.

**Corrections apportées aux propositions métier** (valeurs proposées inexistantes côté Pluxee) :
- `Cadochèque` : métier disait "Autre" -> corrigé en `Autres` (la valeur exacte)
- `La Carte Française` : métier voulait l'identique, mais absente côté Pluxee -> `Autres`
- `Autre` : métier disait "Non-Accessible" -> corrigé en `Non accessible (NSP)` (valeur exacte)

**À signaler au métier** : la perte multi-valeurs (85%) est réelle. Si EmetteurRetenu doit refléter un concurrent précis (pas le premier arbitraire), le mapping est à revoir. `La Carte Française` et `Cadochèque` finissent en `Autres` faute d'équivalent.

## 13. StageName Opportunité (43 stades Glady -> 6 Pluxee)

**Décision** : mappé par déduction. 3 stades couvrent 96% du volume (sûrs) :
- Clôturée - Perdue (56%) -> 6- Fermée Perdue
- Clôturée - Gagnée (27%) -> 5- Fermée Gagnée
- Amorçage des interlocuteurs (13%) -> 1- Nouvelle opportunité

La traîne (~4%) déduite selon la logique de cycle de vente. Défaut : `1- Nouvelle opportunité`.

**Paris à valider avec le métier** :
- Renouvellement en cours (1025) -> 2- Qualification (arbitraire, un renouvellement peut être plus avancé)
- Closing -> 3- Négociation (ou 4- Accord de principe ?)

**Ce que Mario doit savoir** : StageName pilote les validation rules BANT et le reporting pipeline. Mapping déduit sans validation métier formelle (Guillaume absent). À faire confirmer.

## 14. Retours tardifs sur les Leads (champs oubliés au premier passage)

### NumberOfEmployees -> Tranche_d_effectif__c
Champ numérique Glady -> picklist Pluxee. Conversion par formule dans le tMap (pas PicklistMapper, ce n'est pas une correspondance de valeurs mais un découpage en tranches) :
```java
row1.NumberOfEmployees == null ? null :
row1.NumberOfEmployees < 50 ? "moins de 50" :
row1.NumberOfEmployees < 100 ? "50 - 99" :
row1.NumberOfEmployees <= 250 ? "100 - 250" :
row1.NumberOfEmployees <= 500 ? "251 - 500" :
row1.NumberOfEmployees <= 1000 ? "501 - 1000" : "plus de 1000"
```
API names cibles : `moins de 50`, `50 - 99`, `100 - 250`, `251 - 500`, `501 - 1000`, `plus de 1000`.

### InteretProduits__c -> Interessepar__c : NON MAPPÉ
- **0 correspondance** sur 28 valeurs source. Univers incompatibles (produits CSE Glady vs offre Pluxee Meal/Gift).
- Multipicklist -> multipicklist (nécessiterait un translateMulti, non codé).
- Cible = Global Value Set `Interesse_par`, partagé avec Opportunity.Produits__c.
- **Décision** : non mappé pour l'UAT. Nécessite une table de correspondance fournie par le métier si la migration de ce champ est souhaitée.

## 15. Taille_de_l_entreprise__c (Lead) -> Tranche_d_effectif__c

**Source** : picklist `Taille_de_l_entreprise__c` (8 valeurs), PAS `NumberOfEmployees` (abandonné).

**Décision** : mapping approximatif "tranche englobante". Les découpages Glady et Pluxee ne coïncident pas (ex. Glady `51-200` est à cheval sur les tranches Pluxee `50-99` et `100-250`).

| Source Glady | -> Cible Pluxee | Précision |
|---|---|---|
| -10 | moins de 50 | approx (pas de tranche <50 côté Pluxee) |
| 11-50 | moins de 50 | approx |
| 51-200 | 100 - 250 | **à cheval** |
| 201-500 | 251 - 500 | approx |
| 501-1000 | 501 - 1000 | exact |
| 1000+ | plus de 1000 | exact |
| 11-100 | 50 - 99 | **à cheval** |
| 101-500 | 251 - 500 | **à cheval** |

**Ce que Mario doit savoir** : les correspondances marquées "à cheval" sont imprécises. La création de tranches alignées côté Pluxee a été volontairement écartée (ambiguïté trop forte : les bornes Glady ne correspondent à aucune borne Pluxee propre). **À charge du métier/Pluxee de définir les bonnes valeurs cibles** s'ils veulent un découpage exact. En l'état, mapping dégradé assumé pour l'UAT.

## 16. Date_de_cl_ture_estim_e__c (Lead) — date future fixe

**Contexte** : DEUX validation rules Pluxee sur ce champ pour les leads Convertis :
1. "date de clôture estimée requise pour convertir" (champ obligatoire)
2. "la date de clôture estimée doit être ultérieure à la date du jour" (date FUTURE exigée)

La règle #2 rend impossible d'utiliser une date source (ConvertedDate/CreatedDate sont passées → 100% de rejet). Une "date de clôture estimée future" n'a aucun sens pour un lead déjà converti : la règle n'est pas conçue pour de la migration historique.

**Décision (UAT mise à jour)** : champ non mappé (`null`). Depuis l'ajout du filtre `IsConverted=false` (note #24), seuls les leads non convertis sont migrés → `ConvertedDate` est toujours null → l'ancien mapping tombait sur `CreatedDate` (dans le passé) → VR en erreur. La VR ne se déclenche pas si le champ est vide (`NOT(ISBLANK(...))`), donc `null` est la bonne valeur.

**⚠️ Ce que Mario doit savoir** :
- `Date_de_cl_ture_estim_e__c` n'est pas mappé — il reste vide pour tous les leads migrés.
- La VR `DateCloture_requiss_pour_conversion` ne contient pas de check `$User.Bypass_validation_rules__c`. Si un jour il faut mapper ce champ avec des dates passées, il faudra modifier la VR pour ajouter le bypass.

## 17. Fonctions__c (Lead Glady) -> Fonction__c (Lead Pluxee) — valeur par défaut "Autre"

**Contexte** : la VR `Fonction_requis_pour_conversion` exige `Fonction__c` non vide pour les leads au statut "Converti". Le champ Glady `Fonctions__c` (9 valeurs) est mappé vers le GVS Pluxee `Fonction` (55+ valeurs), avec 17 valeurs ajoutées dans MigrWario pour couvrir le mapping métier.

**Décision (UAT)** : `XX_DEFAULT_XX -> Autre` dans le PicklistMapper. Tous les leads dont `Fonctions__c` est vide ou non mappé reçoivent "Autre", qu'ils soient convertis ou non.

**⚠️ Ce que Mario doit savoir** :
- Le défaut "Autre" s'applique à **tous** les leads (pas uniquement les convertis) — pollution mineure mais assumée pour la simplicité.
- Les 17 valeurs ajoutées au GVS `Fonction` doivent être déployées dans l'org cible AVANT la migration prod.
- **En prod, le bypass VR serait plus propre** que le défaut "Autre" sur les leads non convertis.

## 18. Type_societe__c (Lead Pluxee) — valeur par défaut "Entreprise"

**Contexte** : la VR `Type_dentreprise_requis_pour_conversion` exige `Type_societe__c` non vide pour les leads au statut "Converti". **Aucun champ équivalent n'existe côté Glady.**

**Décision (UAT)** : valeur par défaut `Entreprise` posée sur tous les leads (pas de source à mapper, pas de PicklistMapper — valeur en dur dans le tMap).

**⚠️ Ce que Mario doit savoir** :
- Donnée **approximative** : les leads CSE devraient logiquement avoir `CE`, pas `Entreprise`. Aucun moyen fiable de distinguer côté Glady dans le périmètre actuel.
- **En prod, le bypass VR reste la meilleure solution** pour ne pas polluer ce champ avec une valeur par défaut non vérifiée.
- Si le métier fournit une règle de distinction CSE/Entreprise (record type, segment, marché…), le mapping peut être conditionné.

## 19. Marche__c (Lead Glady) — valeur par défaut "Externe" pour les leads convertis

**Contexte** : la VR Pluxee exige `Marche__c` non vide pour convertir un lead. Côté Glady, **99.4% des leads convertis ont Marche__c vide** (38 412 sur 38 639). Le champ n'était pas utilisé opérationnellement côté Glady pour les leads STIM.

**Analyse** : aucun autre champ Glady (TypeLead, TypeSTIM, Raison_du_besoin, Type_de_produit, RecordType, Segmentation…) ne permet de déduire Interne vs Externe de façon fiable — les mêmes valeurs apparaissent dans les deux marchés.

**Décision (UAT)** : défaut `Externe` injecté dans le tMap quand `Marche__c` est vide, pour les leads convertis. La cascade Marche → Famille_de_produit__c utilise cette valeur par défaut.

**⚠️ Ce que Mario doit savoir** :
- **38 412 leads** reçoivent `Externe` par défaut — donnée approximative (distribution observée sur les renseignés : 86% Externe / 13% Interne).
- ~30 leads Glady convertis avec Marche = `Interne` sont correctement mappés (pas de défaut appliqué).
- **Le métier doit corriger la source Glady** avant la migration prod si la distinction Interne/Externe est importante pour le reporting.
- **En prod, le bypass VR reste la solution la plus propre** pour ne pas polluer ce champ.

## 20. Bypass Validation Rules — activé sur le user de migration (Lead)

**Contexte** : 3 703 rejets Lead en UAT causés par des VR Pluxee (date de première commande, format téléphone, SIRET, type de société, longueur nom/prénom). Ces VR protègent la saisie manuelle mais bloquent la migration de données Glady qui n'ont jamais été soumises à ces règles.

**Décision (UAT)** : `Bypass_validation_rules__c = true` activé sur le user de migration dans MigrWario. Les VR custom ne bloquent plus l'import.

**⚠️ Ce que Mario doit savoir** :
- Le bypass masque des problèmes de qualité de données (téléphones mal formatés, SIRET invalides, champs obligatoires vides).
- **Pour améliorer la qualité des données en prod** : désactiver le bypass, relancer la migration, et traiter les rejets un par un (nettoyage source Glady ou enrichissement du mapping).
- Les contraintes plateforme (ConvertedDate < CreatedDate, restricted picklists, STRING_TOO_LONG) ne sont **pas** couvertes par le bypass — elles nécessitent un fix dans le tMap ou un nettoyage de données.

## 21. Raison_du_statut__c (Lead Pluxee) — défaut "Mauvaise qualification" conditionné sur le statut

**Contexte** : la VR Pluxee exige `Raison_du_statut__c` non vide pour les leads au statut "Trash (Non-Converti)" ou "Remarketing (Non-Converti)". Le champ source Glady `CauseNonQualifie__c` est souvent vide, même pour les leads non qualifiés.

**Décision (UAT)** : le défaut "Mauvaise qualification" est appliqué **uniquement** quand le statut cible Pluxee est "Trash (Non-Converti)" ou "Remarketing (Non-Converti)" ET que `CauseNonQualifie__c` est vide. Les autres leads gardent `Raison_du_statut__c` vide. Pas de `XX_DEFAULT_XX` dans le CSV — la logique est entièrement dans le tMap.

**⚠️ Ce que Mario doit savoir** :
- "Mauvaise qualification" est une approximation — la vraie raison de non-conversion n'est pas connue pour ces leads.
- Si le métier fournit une règle plus fine, la condition dans le tMap peut être ajustée.

## 22. StageName Opportunity — mapping métier + filtrage des stages "A supprimer"

**Contexte** : 47 stages Glady réduits à 10 stages migrés, mappés vers les 6 stages Pluxee. Le mapping provient du fichier métier (`Oppo/mappingMetier.xlsx`). Les stages marqués "A supprimer" par le métier sont exclus de la requête source.

**Filtre SOQL appliqué** :
```
AND StageName IN ('Nouvelle opportunité','Renouvellement en cours','RDV fait','R1 pris','R1 annulé','Suivi R1','Closing','R2 et plus','Qualification','Négociation','Clôturée - Gagnée','Clôturée - Perdue')
```

**Mapping CSV** :

| Glady | → Pluxee |
|---|---|
| Nouvelle opportunité | 1- Nouvelle opportunité |
| R1 pris | 1- Nouvelle opportunité |
| R1 annulé | 1- Nouvelle opportunité |
| RDV fait | 2- Qualification |
| Suivi R1 | 2- Qualification |
| Qualification | 2- Qualification |
| R2 et plus | 3- Négociation |
| Négociation | 3- Négociation |
| Closing | 4- Accord de principe |
| Renouvellement en cours | 5- Fermée Gagnée |

**⚠️ Ce que Mario doit savoir** :
- **Clôturée - Gagnée / Perdue réintégrées** : le fichier métier les marquait "A supprimer" mais fournissait un mapping (5- Fermée Gagnée / 6- Fermée Perdue). Arbitrage migration : elles sont migrées (66k oppos, 83% du volume). Si le métier confirme l'exclusion, retirer ces 2 valeurs du filtre SOQL.
- **~14k opportunités restent exclues** (Amorçage 10 415, BANT complété 442, Solutions présentés 252, Tarification envoyée 378, etc.). Voulu par le métier.
- Le métier avait fourni des noms de stages (1. BANT complété, etc.) qui ne correspondent pas aux API Names Pluxee — la correspondance a été faite par position (1→1, 2→2, etc.).
- `Renouvellement en cours` → `5- Fermée Gagnée` : décision métier, ces oppos sont considérées comme gagnées.

## 23. Permission Set "Data Migration Users" — configuration du user de migration

**Contexte** : plusieurs VR et contraintes Pluxee bloquent l'import de données migrées. Un Permission Set centralisé regroupe tous les bypass nécessaires.

**Permission Set** : `Data Migration Users` (déployé dans MigrWario, UAT Wario et Git)

**Contenu** :
- `Bypass_validation_rules__c` (User custom field) — bypass des VR custom Lead/Opportunity
- `Bypass duplicated rules` (User custom field) — bypass des Duplicate Rules Lead
- `EditClosedOpportunity` (Custom Permission) — permet de modifier les opportunités fermées (VR `BlockClosedOppModification`)

**⚠️ Ce que Mario doit savoir** :
- Ce Permission Set doit être assigné au user de migration **avant** chaque run.
- **En prod, le retirer après la migration** pour restaurer les contrôles de saisie.
- Si de nouvelles VR bloquent l'import, ajouter les bypass nécessaires dans ce PS plutôt que de créer un nouveau.

## 24. Leads — filtrage sur IsConverted=false uniquement

**Décision (UAT)** : seuls les leads non convertis (`AND IsConverted=false`) sont migrés. Les leads convertis ne sont pas accessibles dans Salesforce (ils deviennent read-only et masqués derrière le Contact/Account/Opportunity convertis), donc les migrer n'apporte rien.

**⚠️ Ce que Mario doit savoir** :
- Les leads convertis existent côté Glady mais ne sont **pas** dans le périmètre de migration.
- Les données des leads convertis sont accessibles via leurs Account/Contact/Opportunity convertis (qui eux sont migrés séparément).
