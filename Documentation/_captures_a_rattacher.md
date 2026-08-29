# Captures d'écran — où les rattacher dans Confluence

Fichier de travail, **à ne pas publier**. Il indique quelle image attacher à quelle page.

Dans chaque page HTML, l'emplacement est marqué par un encadré gris commençant par 📎.
Une fois l'image attachée dans Confluence, remplacer l'encadré par la macro image.

## Captures indispensables (4)

| Page Confluence | Section | Fichier source | Légende à reprendre |
|---|---|---|---|
| **1. Architecture & périmètre** | 1.1 Les deux orgs | `Contents/Screenshots/Capture d'écran 2026-08-28 145811.png` | Arborescence des métadonnées Salesforce dans le référentiel Talend — les deux connexions et les objets rattachés à chacune. |
| **1. Architecture & périmètre** | 1.4 Ordre interne du job Oppo/Devis | `Contents/Screenshots/Job MigrationPluxeeOppoDevis.png` | Vue d'ensemble du job Opportunités/Devis. Les huit blocs correspondent aux huit sous-jobs ; les liens verts matérialisent l'enchaînement `OnSubjobOk`. |
| **1. Architecture & périmètre** | 1.6 Outils et versions | `Contents/Screenshots/Capture d'écran 2026-08-28 145434.png` | Le référentiel Talend — les deux jobs, les contextes, et la routine `PicklistMapper` sous *Code > Routines globales*. |
| **2. Process Talend** | 2.1 Anatomie d'un sous-job | `Contents/Screenshots/Capture d'écran 2026-08-28 145140.png` | Le job Lead dans son ensemble. Le bloc du haut est le sous-job Lead ; les deux blocs inférieurs le répètent à l'identique. À gauche, les branches `tPrejob` et `tPostjob`. |

## Captures facultatives

Issues de la documentation générée par Talend (dossier `pictures/` de chaque job).
À ajouter seulement si tu veux illustrer davantage — la doc se tient sans elles.

| Page | Section | Fichier | Intérêt |
|---|---|---|---|
| 2. Process Talend | 2.1 | `MigrationPluxeeLead_v2/pictures/tMap_1.png` | Détail d'un tMap : entrées à gauche, lookups au centre, sortie mappée à droite. Rend concret le schéma en texte. |
| 2. Process Talend | 2.3 | `MigrationPluxeeOppoDevis/pictures/tMap_13.png` | Le tMap des Devis, avec ses trois lookups en jointure interne. Bon exemple du mécanisme décrit. |
| 1. Architecture | 1.4 | `MigrationPluxeeLead_v2/pictures/MigrationPluxeeLead_v2_0.1.png` | Vue d'ensemble du job Lead générée par Talend — alternative plus nette à la capture d'écran. |

## Note

Les captures livrées ne montrent que des schémas de jobs et des arborescences de référentiel :
aucune donnée personnelle, aucun nom de client, aucun identifiant de connexion n'y est lisible.
Elles peuvent être publiées telles quelles.

Deux d'entre elles font apparaître le nom technique de la connexion cible dans l'arbre du référentiel.
Les pages parlent d'« org de destination » ; si tu veux l'alignement parfait texte/image,
renomme la connexion dans Talend avant de refaire la capture — sinon, laisse tel quel,
c'est un identifiant technique sans ambiguïté pour qui ouvre le Studio.
