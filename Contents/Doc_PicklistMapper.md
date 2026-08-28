# PicklistMapper — Documentation technique

**Version courante : 5.0 (17 juillet 2026) · Routine Talend partagée · Projet migration Glady → Pluxee (équipe Wario)**

## 1. Rôle

Routine Java Talend qui traduit les valeurs de picklist d'une org Salesforce source vers une org cible, pilotée par un **fichier CSV externe**. Objectif : aucune règle de correspondance dans les jobs — tout est modifiable en éditant le CSV, sans toucher à Talend. Multi-objet (Lead, Account, Contact, …), utilisable par toute l'équipe.

Ce que la routine ne fait **pas** (décisions d'architecture) :
- Elle ne crée jamais de nouvelles valeurs de picklist dans la cible. L'enrichissement d'une picklist cible est une opération de **métadonnées** à faire avant la migration (déploiement), alimentée par le fichier d'audit.
- Elle ne normalise ni casse ni accents : `Contacte` et `Contacté` sont deux valeurs distinctes. Chaque variante présente dans les données = une ligne du CSV.

## 2. Installation

1. Référentiel Talend > `Code` > `Routines` > créer (ou ouvrir) `PicklistMapper`, coller le contenu de `PicklistMapper.java` (garder `package routines;`), sauvegarder.
2. Créer deux variables de contexte dans le job : `picklist_mapping_path` (chemin du CSV de correspondances) et `picklist_unmapped_path` (chemin du fichier d'audit produit en fin de job).

## 3. Format du fichier CSV

- **Encodage UTF-8, séparateur `;`, une ligne d'en-tête** (ignorée).
- ⚠️ **Lecture PAR POSITION, pas par nom d'en-tête.** Ne jamais réordonner les 4 premières colonnes. Des colonnes libres (commentaires, volumes…) peuvent être ajoutées **après** la 4e ; elles sont ignorées.

```
object;source_field;source_value;target_value;target_field
Lead;Status;Disqualifié;Trash (Non-Converti)
Lead;Status;Marketing Automation;Lead
Account;Fonctions__c;Direction;DG
Account;Fonctions__c;XX_DEFAULT_XX;Autre
Lead;CauseNonQualifie__c;XX_DEFAULT_XX;
```

| Colonne | Contenu | Règles |
|---|---|---|
| `object` | API name de l'objet **source** (`Lead`, `Account`…) | jamais le label ; convention d'équipe : API name |
| `source_field` | API name du champ **source** | toujours le champ source, même si la cible porte un autre nom (nécessaire pour les cas plusieurs-sources → une-cible) |
| `source_value` | valeur telle que stockée dans la source (API name de valeur) | sensible casse/accents ; `XX_DEFAULT_XX` est **réservé** (voir §5) |
| `target_value` | valeur écrite en cible (API name de valeur) | vide = « vider le champ » (retour `null`) |

**Construire le CSV depuis les données réelles, jamais depuis la picklist** : `SELECT champ, COUNT(Id) FROM objet GROUP BY champ` sur la source. Les données contiennent souvent plus de valeurs que la picklist active (valeurs historiques, labels stockés comme valeurs après renommage, variantes d'accents). Cas vécu : `Lead.Status`, 8 valeurs actives, **28 valeurs distinctes stockées**.

## 4. API

| Méthode | Usage |
|---|---|
| `load(path)` | Charge le CSV en **remplaçant** tout (correspondances, défauts, compteurs). À appeler dans `tPreJob > tJava`. Lève une exception si le fichier est illisible (le job s'arrête : comportement voulu). |
| `loadMore(path)` | Charge un CSV **en plus** de l'existant (organisation « un fichier par objet »). |
| `translate(object, field, value)` | Traduit. Retourne la cible, ou `null` (voir résolution §5). |
| `translate(object, field, value, defaultValue)` | Idem avec filet ultime si ni correspondance ni défaut CSV. |
| `dumpUnmapped(path)` | Écrit le fichier d'audit (§6) et affiche le résumé par champ en console. À appeler dans `tPostJob > tJava`. |
| `getSummary()` | Retourne le résumé par champ (String) sans écrire de fichier. |
| `setDebug(boolean)` | Log console de **chaque** traduction : `[objet][champ] valeur avant : xxx, valeur apres : yyy (origine)`. Réserver aux petits volumes de test. |
| `DEFAULT_KEY` | Constante publique = `"XX_DEFAULT_XX"`. |

Exemple d'expression tMap : `PicklistMapper.translate("Lead", "Status", row1.Status)`

## 5. Valeur par défaut et ordre de résolution

Une ligne dont `source_value` vaut `XX_DEFAULT_XX` définit le **défaut du champ** : toute valeur source absente des lignes listées reçoit cette cible. `target_value` vide sur cette ligne = « défaut : vider le champ ».

Ordre de résolution de `translate` :
1. **Correspondance exacte** `object|field|value` (une cible vide est retournée comme `null`).
2. **Défaut du champ déclaré dans le CSV** (`XX_DEFAULT_XX`) — *le CSV prime toujours sur l'argument*.
3. **Argument `defaultValue`** de la méthode (filet ultime).
4. **`null`** + enregistrement en « unmapped ».

Les cas 2 et 3 sont enregistrés en « defaut » dans l'audit (jamais silencieux).

**Collision** : si une valeur source réelle vaut littéralement `XX_DEFAULT_XX`, un avertissement est émis en console (une seule fois par champ) et la valeur est traitée comme une valeur normale (elle ne matche pas la ligne défaut, qui est stockée à part).

**Stratégie d'usage (règle d'équipe)** : le défaut est un **choix explicite par champ**, pas un réflexe.
- Champ porteur de sens métier (Status, raisons, fonctions…) → **pas de ligne défaut** : chaque valeur inconnue doit remonter en « unmapped » et être tranchée.
- Champ secondaire où « Autre » est une réponse honnête → ligne défaut assumée, auditée via le type « defaut ».
- Ne poser un défaut qu'après avoir fait le `GROUP BY` du champ.
- Champ cible **restricted** : toute valeur produite (y compris le défaut) doit exister dans le value set cible, sinon rejet à l'INSERT.

## 6. Fichier d'audit (`dumpUnmapped`)

Format : `object;field;source_value;type` avec `type ∈ {unmapped, defaut}`.
- `unmapped` = valeur rencontrée sans correspondance ni défaut (résultat `null`). À traiter : ajouter une ligne au CSV, ou décider un défaut, ou créer la valeur cible (métadonnées).
- `defaut` = valeur passée par un défaut (CSV ou argument). À surveiller : un volume anormal peut signaler une valeur mal orthographiée ou un cas métier oublié.

Les entrées sont dédupliquées (valeurs **distinctes**, pas un décompte de lignes traitées). Le résumé console de fin de job donne, par champ, le nombre de valeurs distinctes inconnues et passées par le défaut.

## 7. Branchement Talend type

```
tPreJob ──OnComponentOk──▶ tJava :
    PicklistMapper.load(context.picklist_mapping_path);
    // PicklistMapper.setDebug(true);   // test uniquement

tMap (expressions de sortie) :
    PicklistMapper.translate("Lead", "Status", row1.Status)

tPostJob ──▶ tJava :
    PicklistMapper.dumpUnmapped(context.picklist_unmapped_path);
```

## 8. Pièges connus

- **État `static` partagé par JVM** : un job seul, aucun souci (chaque exécution TOS = une JVM). Avec des sous-jobs `tRunJob` dans la même JVM, un `load()` écrase ce qu'un autre a chargé → charger une seule fois au parent, ou utiliser `loadMore`.
- **API names, pas labels** : SOQL lit et l'INSERT écrit l'API name des valeurs. Les labels ne matchent pas.
- **Séparateur `;`** : une valeur contenant `;` casserait le parsing. Aucun cas rencontré à ce jour ; changer de séparateur si ça arrive.
- **Encodage** : CSV en UTF-8 obligatoire ; un CSV enregistré en ANSI/latin-1 fait échouer silencieusement les jointures sur les valeurs accentuées.
- **Ne jamais réordonner les 4 premières colonnes** (lecture par position).


## 9. Champ cible explicite (`target_field`) — dépendances et cibles multiples

⚠️ **Cette section ne décrit pas du mapping de valeurs.** Les lignes avec `target_field`
ne traduisent pas une valeur vers son équivalent : elles expriment une **règle structurelle**
de l'org cible — typiquement la valeur qu'un champ *parent* doit porter pour qu'une valeur
de champ *enfant* soit insérable. On réutilise le même fichier parce que le déclencheur est
le même (une valeur source), mais l'intention est différente : ce n'est pas « X devient Y »,
c'est « quand la source vaut X, ce champ-là doit valoir Y pour que l'insert passe ».

### Format
La 5e colonne `target_field` indique l'API name du champ **cible** quand il diffère du
champ source. Vide (ou absente) = la cible porte le même nom que la source (comportement
historique, tous les CSV antérieurs restent valides).

### Cas 1 — cascade de picklists dépendantes
Dans MigrWario, `Famille_de_produit__c` contrôle `Produit__c`, qui contrôle `Marche__c`.
Les valeurs `STIMI`/`STIME` de `Marche__c` ne sont insérables que si `Produit__c = CCI`
et `Famille_de_produit__c = Gift Incentive`. Une seule valeur source (`Marche__c` côté
Glady) doit donc alimenter trois champs cibles :

```
object;source_field;source_value;target_value;target_field
Lead;Marche__c;Interne;STIMI;
Lead;Marche__c;Interne;CCI;Produit__c
Lead;Marche__c;Interne;Gift Incentive;Famille_de_produit__c
```

Appels dans le tMap :
```java
// champ Marche__c en sortie
PicklistMapper.translate("Lead","Marche__c",row1.Marche__c)
// champ Produit__c en sortie
PicklistMapper.translateTo("Lead","Marche__c",row1.Marche__c,"Produit__c")
// champ Famille_de_produit__c en sortie
PicklistMapper.translateTo("Lead","Marche__c",row1.Marche__c,"Famille_de_produit__c")
```
Une valeur source non listée retourne `null` sur les trois champs : aucune cascade
partielle n'est posée, et le trio reste cohérent.

### Cas 2 — une source alimentant plusieurs cibles indépendantes
Même mécanisme sans notion de dépendance (ex. un champ d'origine alimentant plusieurs
champs de qualification). Une ligne par champ cible, avec le `target_field` correspondant.

### Limite connue
Les lignes d'une même cascade sont **indépendantes** : rien dans le fichier ne garantit
qu'elles soient maintenues ensemble. Modifier la ligne enfant sans les lignes parentes
produira un rejet Salesforce peu explicite (`INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST`).
Règle d'équipe : toute modification d'une cascade se fait sur les 3 lignes à la fois.
