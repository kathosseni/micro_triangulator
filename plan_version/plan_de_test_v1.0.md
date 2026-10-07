# Plan de test

## 1. Objectif et périmètre

Ce document décrit la stratégie de test du microservice `Triangulator`, avant
toute implémentation. Le `Triangulator` :
- expose `GET /triangulation/{pointSetId}` (spec `triangulator.yml`)
- interroge en interne `PointSetManager` via `GET /pointset/{pointSetId}`
  (spec `point_set_manager.yml`) pour récupérer les données à trianguler
- calcule une triangulation et renvoie le résultat au format binaire `Triangles`

`PointSetManager` est **simulé** (stub/mock), dans tous nos tests à la frontière HTTP du `Triangulator`.

## 2. Décisions de conception

### 2.1 Validation du 'pointSetId'

**Validation locale uniquement (format UUID), avant tout appel
réseau.**
 
- Si `pointSetId` n'est pas un UUID syntaxiquement valide → `400` immédiat,
  **sans contacter `PointSetManager`**.
- Si le format est valide → on appelle `PointSetManager`. À ce stade, il ne
  peut plus renvoyer de `400` dans notre flux : ce cas de sa spec n'est donc
  jamais exploité côté `Triangulator`.
Conséquence testable : un test sur un ID mal formé doit aussi vérifier que
le client HTTP vers `PointSetManager` n'a **pas été appelé**
(`mock.assert_not_called()`), pas seulement que le code retour est `400`.

Conséquence testable : un test sur un ID mal formé doit aussi vérifier que
le client HTTP vers `PointSetManager` n'a **pas été appelé**
(`mock.assert_not_called()`), pas seulement que le code retour est `400`.

### 2.2 Mapping des erreurs de 'pointSetId'

| Réponse de `PointSetManager` (simulée) | Réponse du `Triangulator` |
|---|---|
| `pointSetId` invalide (détecté localement) | `400` — sans appel réseau |
| `404` | `404` |
| `503` ou erreur réseau / timeout | `503` |
| `200` + `PointSet` valide, triangulation OK | `200` + `Triangles` |
| `200` + `PointSet` valide, triangulation impossible (trop peu de points, points colinéaires...) | `500` |
| `200` + payload binaire incohérent/corrompu | `500` (traité comme une erreur interne d'interprétation des données) |

### 2.3 Design à tester

- La logique de décodage/encodage binaire est une fonction pure,
  indépendante de Flask et du réseau.
- L'algorithme de triangulation est une fonction pure prenant un ensemble
  de points et renvoyant des triangles (ou une erreur), sans dépendance
  externe.
- L'appel à `PointSetManager` est isolé derrière une interface/fonction
  dédiée (ex. `PointSetClient.get(point_set_id)`), injectable dans les
  tests via un double.
- La route Flask réalise les actions suivantes : valider → appeler le client → décoder →
  trianguler → encoder → répondre. Aucune de ces étapes n'est testée uniquement à ce niveau ; chacune a ses propres tests unitaires en amont.

## 3. Niveau de test

### 3.1 Structure

| Niveau | Ce qui est vérifié | Outils |
|---|---|---|
| Unitaire — format binaire | Encodage/décodage `PointSet` et `Triangles` | `pytest` |
| Unitaire — validation | Validation du `pointSetId` (UUID) | `pytest`, `@pytest.mark.parametrize` |
| Unitaire — algorithme | Correction de la triangulation | `pytest`, éventuellement propriétés |
| API / composant | Comportement HTTP du `Triangulator`, `PointSetManager` mocké | `pytest`, client de test Flask, `unittest.mock` |
| Performance | Temps de calcul sous charge | `pytest`, marqueur `performance` |
| Couverture | Lignes/branches réellement exercées par les tests | `coverage.py` |
| Qualimétrie | Style, conventions, documentation du code | `ruff`, `pdoc3` |
| Sécurité | Robustesse face à des entrées hostiles | `pytest` (cas dédiés) |
| Intégration / système | Comportement bout-en-bout avec un vrai `PointSetManager` | `pytest` (optionnel, voir §12) |
| Acceptation | Critères formulés du point de vue du besoin | recoupe les niveaux ci-dessus |

### 3.2 Application du TDD

Les tests étant écrits avant l'implémentation : l'étape suivante correspond à l'état "Red" du cycle Red → Green → Refactor pour l'ensemble des catégories ci-dessous — tests présents, en échec, sans code applicatif derrière. Les séances suivantes font passer ces tests au vert puis refactorisent, en autorisant les tests eux-mêmes à évoluer si la réalité de l'implémentation le justifie.

Ce cycle s'applique naturellement aux catégories dont le comportement
attendu est déductible des specs *avant* tout code : tests unitaires
(§4, §5, §6) et tests API/composant (§7), puisque le contrat HTTP et
les formats binaires sont entièrement connus à l'avance.

Il s'applique plus difficilement, ou seulement partiellement, à
certaines catégories :
- **Performance (§8)** : un budget de temps ne peut être fixé de façon
  réaliste qu'après avoir observé une première implémentation
  fonctionnelle. Les tests seront donc écrits en amont (structure,
  marqueur `performance`, scénarios de charge) mais avec des seuils
  volontairement larges, resserrés une fois une mesure de référence
  obtenue.
- **Sécurité (§11)** : les cas d'entrées hostiles sur le format ou sur
  le `pointSetId` sont déductibles à l'avance et suivent le TDD
  normalement ; en revanche, certains cas (ex. limites mémoire
  réellement observées) peuvent être affinés après une première
  implémentation.
- **Qualimétrie (§10)** : par nature continue, appliquée au fil de
  l'écriture du code plutôt qu'en amont sous forme de tests.
- **Intégration/système (§12)** : dépend de l'option retenue ; si un
  `PointSetManager` factice est développé, ses propres tests suivront
  le même principe TDD, indépendamment du `Triangulator`.

Appliquer un TDD strict et uniforme à toutes les catégories serait artificiel pour certaines d'entre elles, et ce plan préfère le documenter plutôt que de le prétendre.

## 4. Test Unitaires - Format binaire

### 4.1 Décodage de `PointSet`

Cas nominaux :
- 0 point (en-tête seul, 4 bytes)
- 1 point
- N points (petit N, ex. 5)
- coordonnées négatives, nulles, très grandes/petites (limites du `float`)

Cas d'erreur :
- payload plus court que l'en-tête (moins de 4 bytes)
- en-tête annonçant N points mais payload trop court (troncature au milieu
  d'un point, ou au milieu d'un `float`)
- payload plus long que ce qu'annonce l'en-tête (bytes en trop, à définir :
  tolérer ou rejeter ?)
- payload vide

### 4.2 Encodage de `PointSet`

- round-trip : `decode(encode(points)) == points` pour plusieurs jeux de
  points (0, 1, N points)
- vérification manuelle de la taille produite (`4 + 8*N` bytes) et de
  l'ordre des bytes (endianness) sur un cas simple, valeur attendue
  calculée à la main

### 4.3 Décodage de `Triangles`

Cas nominaux :
- 0 triangle (sommets présents, section triangles vide)
- 1 triangle, N triangles

Cas d'erreur :
- indices de sommet hors bornes (référence un sommet qui n'existe pas
  dans la partie `PointSet`)
- payload tronqué dans la section triangles
- nombre de triangles annoncé incohérent avec la taille du payload

### 4.4 Encodage de `Triangles`

- round-trip `decode(encode(triangles)) == triangles`
- vérification de la structure produite (taille exacte, position des
  deux sections)

## 5. Test Unitaires - Validation du pointSetId

Fonction pure, aucun mock nécessaire.

- UUID valide (format standard avec tirets) → accepté
- chaîne vide → rejeté
- chaîne quelconque non-UUID (`"abc123"`) → rejeté
- UUID avec mauvais nombre de caractères → rejeté
- UUID avec tirets mal placés/absents → rejeté
- `None` ou type inattendu → rejeté
- casse (majuscules/minuscules) → comportement à définir et documenter

## 6. Test Unitaires - Algo de triangularisation

Cas nominaux :
- ensemble de 3 points (cas trivial : un seul triangle possible)
- ensemble de points en position générale (N ≥ 4)
- vérification de propriétés générales plutôt que de résultats exacts
  ligne à ligne, par exemple :
  - tous les indices de sommets référencés dans les triangles existent
    dans le `PointSet` d'entrée
  - chaque point du `PointSet` d'entrée est utilisé dans au moins un
    triangle
  - aucun triangle dégénéré (aire nulle)

Cas limites / erreurs :
- 0, 1 ou 2 points → pas assez pour trianguler, erreur attendue
- points tous colinéaires → triangulation impossible, erreur attendue
- points dupliqués (même coordonnées) → comportement à définir et
  documenter (ignorer les doublons ? erreur ?)

## 7. Test API/composant

Utilisation du client de test Flask ; `PointSetManager` simulé via un
double injecté (mock/stub) sur le client HTTP interne.

Ces tests valident le comportement du `Triangulator` **isolé**, en
supposant que `PointSetManager` respecte sa spec. Ce ne sont pas des
tests d'intégration au sens strict (voir §13) : ils ne font intervenir
aucun vrai `PointSetManager`.

| # | Scénario | `PointSetManager` simulé | Réponse attendue |
|---|---|---|---|
| 1 | `pointSetId` mal formé | non appelé (`assert_not_called`) | `400` |
| 2 | UUID valide, ID inconnu | `404` | `404` |
| 3 | UUID valide, service indisponible | `503` / timeout | `503` |
| 4 | UUID valide, `PointSet` valide et triangulable | `200` + payload valide | `200` + `Triangles` corrects |
| 5 | UUID valide, `PointSet` insuffisant (< 3 points) | `200` + payload valide | `500` |
| 6 | UUID valide, `PointSet` colinéaire | `200` + payload valide | `500` |
| 7 | UUID valide, payload binaire corrompu renvoyé | `200` + payload invalide | `500` |

Pour le scénario 4, vérifier aussi :
- le `Content-Type` de la réponse (`application/octet-stream`)
- que le contenu binaire décodé correspond à une triangulation valide
  des points fournis (via les propriétés du §6, pas une comparaison
  octet-à-octet fragile)

## 8. Tests de performance

Séparés du reste via `@pytest.mark.performance`, exclus de `make unit_test`,
exécutés uniquement par `make perf_test`.

- décodage/encodage `PointSet` sur un grand nombre de points (ex.
  10k, 100k) — mesure du temps, budget à définir
- triangulation sur un grand nombre de points — mesure du temps, budget
  à définir
- répétition des mesures (plusieurs exécutions) plutôt qu'une seule
  valeur, pour limiter le bruit

Ces budgets sont volontairement larges dans un premier temps ; ils seront
ajustés une fois une première implémentation disponible pour obtenir une
mesure de référence.

## 9. Couverture 

- Objectif : couverture de lignes et de branches proche de 100 % sur le
  code applicatif (hors code de configuration/lancement).
- La couverture est un indicateur de zones non exercées, pas un objectif
  en soi : un test ajouté uniquement pour augmenter un pourcentage sans
  assertion pertinente n'est pas acceptable.
- Utilisation de `coverage run --branch -m pytest` pour la mesure de
  branches, en particulier utile sur le mapping d'erreurs du §7 et les
  cas limites de décodage du §4.

## 10. Qualimétrie

Pas exécutée via `pytest`, mais fait partie intégrante de la validation
du projet et doit donc être planifiée au même titre que les tests.

- **Lint** : `ruff check` doit passer sans diagnostic sur l'ensemble du
  code applicatif (`make lint`). Les règles de base sont celles du
  `pyproject.toml` fourni ; possibilité d'en ajouter (jamais d'en
  retirer).
- **Documentation** : toutes les fonctions/classes publiques doivent
  avoir une docstring conforme aux règles `D` de `ruff`
  (`ruff check --select D`), avec au minimum : description, `Args`,
  `Returns`, `Raises` le cas échéant — en particulier pour les
  fonctions de décodage/encodage binaire et l'algorithme de
  triangulation, où les préconditions/erreurs sont importantes à
  documenter.
- **Génération de la doc** : `pdoc3` (`make doc`) doit s'exécuter sans
  erreur et produire une documentation HTML exploitable.
- **Formatage** : `ruff format --check` peut être utilisé en complément,
  sans être une exigence bloquante de l'énoncé.

Ce contrôle est vérifié à chaque exécution de `make lint` et `make doc`,
pas seulement à la fin du projet, pour éviter une dette de qualité qui
s'accumule.

## 11. Tests de sécurité

Le périmètre de sécurité est limité (pas d'authentification, pas de
base de données propre au `Triangulator`), mais deux surfaces
méritent des tests dédiés, distincts des tests de robustesse du §4 et
§5 même s'ils s'appuient sur les mêmes mécanismes :

- **Entrées hostiles sur `pointSetId`** : chaînes très longues,
  caractères spéciaux, tentatives d'injection dans l'URL — doivent
  être rejetées par la validation UUID (§2.1, §5) avant tout
  traitement ou appel réseau.
- **Payloads binaires conçus pour épuiser les ressources** : en-tête
  `PointSet`/`Triangles` annonçant un nombre de points ou de triangles
  extrêmement élevé (proche de la valeur max d'un `unsigned long`) dans
  le but de provoquer une allocation mémoire excessive ou un déni de
  service. Le décodeur doit rejeter ou borner ce cas plutôt que de
  tenter d'allouer la structure correspondante.
- **Non-divulgation d'informations internes** : les réponses d'erreur
  (`400`, `404`, `500`, `503`) ne doivent contenir ni trace de la pile
  d'appel, ni chemin de fichier, ni détail d'implémentation — seulement
  un `code` et un `message` conformes au schéma `Error` des specs.

## 12. Tests d'intégration et système

**Constat de périmètre** : `PointSetManager` n'est pas implémenté dans
le cadre de ce TP. Une intégration réelle entre `Triangulator` et un
`PointSetManager` fonctionnel n'est donc pas testable telle quelle.

Deux options, à trancher avant le rendu n°2 :

- **Option retenue par défaut** : considérer que le contrat
  `point_set_manager.yml` fait foi, et que les tests du §7 (avec double
  conforme à ce contrat) constituent la meilleure approximation possible
  de l'intégration dans le périmètre du TP. Documenter explicitement
  cette limite plutôt que de la passer sous silence.
- **Option renforcée (si le temps le permet)** : développer un
  `PointSetManager` factice minimal (petit serveur Flask, stockage en
  mémoire, conforme à sa spec OpenAPI) et l'utiliser dans un test
  d'intégration réel — un vrai processus HTTP interrogé par le
  `Triangulator`, sans mock. Ce test viendrait compléter, et non
  remplacer, les tests de composant du §7.

Un test système complet (`Client` → `PointSetManager` → `Triangulator`)
suivrait le workflow décrit dans l'énoncé (enregistrement, puis
triangulation) et ne peut donc être envisagé que si l'option renforcée
ci-dessus est mise en place.

## 13. Tests d'acceptation

Formulés du point de vue du besoin métier (le `Client`), indépendamment
des détails d'implémentation. Chaque critère ci-dessous doit pouvoir se
retrouver dans un test concret des sections précédentes.

- Un utilisateur envoie un ensemble de points valides et obtient en
  retour une triangulation exploitable de ces points (§7, scénario 4).
- Un utilisateur interrogeant un identifiant de point set inexistant
  reçoit une erreur claire et explicite, sans ambiguïté sur la cause
  (§7, scénario 2).
- Le service reste utilisable (répond dans un délai raisonnable) sur un
  volume de points représentatif d'un usage réel, pas seulement sur des
  cas triviaux (§8).
- Le service ne plante pas et répond de façon contrôlée face à des
  données mal formées, que ce soit dans l'identifiant ou dans les
  données du `PointSet` récupéré (§4, §5, §11).

## 14. Points de réflexion à décider en cours d'implémentation

- Comportement exact en cas de payload `PointSet` plus long que prévu
  (bytes surnuméraires ignorés ou erreur ?)
- Comportement en cas de points dupliqués dans l'algorithme
- Sensibilité à la casse du `pointSetId`
- Valeur exacte des budgets de performance

Ces points seront documentés au fil de l'eau et repris dans `RETEX.md`.