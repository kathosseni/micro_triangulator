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
le `PointSetManager` n'est **pas sollicité**, pas seulement que le code
retour est `400`. (La façon dont ce non-appel est vérifié en pratique —
quel outil, quelle API — relève de l'écriture des tests, pas de ce plan.)

### 2.2 Mapping des erreurs de 'pointSetId'

Chaque ligne précise non seulement le code HTTP renvoyé, mais aussi le
`code` machine du corps `Error` (schéma des specs) et le module qui
origine l'erreur — exigence explicite du retour du professeur sur les
§4/§5/§6. La taxonomie d'exceptions complète est définie en §2.4.

| Situation | Exception levée, module d'origine | HTTP + `code` renvoyé |
|---|---|---|
| `pointSetId` invalide (détecté localement) | `InvalidPointSetIdError` — `triangulator.validation` | `400` / `INVALID_POINT_SET_ID` |
| `PointSetManager` répond `404` | — (relayé tel quel) | `404` / `NOT_FOUND` |
| `PointSetManager` répond `503` ou timeout/erreur réseau | — (relayé tel quel) | `503` / `POINT_SET_MANAGER_UNAVAILABLE` |
| `PointSet` reçu valide, triangulation OK | — | `200` + `Triangles` |
| Payload `PointSet` reçu corrompu/incohérent | `BinaryFormatError` — `triangulator.binary` | `500` / `INVALID_POINT_SET_DATA` |
| `PointSet` valide mais triangulation impossible | `TriangulationError` (ou sous-classe) — `triangulator.geometry` | `500` / `TRIANGULATION_FAILED` |

Distinguer `INVALID_POINT_SET_DATA` de `TRIANGULATION_FAILED` permet de
savoir, à la lecture d'une erreur `500`, si la cause est un problème de
format binaire ou un problème géométrique — une information perdue si
les deux cas étaient fondus sous un seul code `500` générique.

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

### 2.4 Taxonomie des erreurs

Décision de conception, à fixer avant d'écrire la moindre ligne de code
applicatif : chaque fonction qui peut échouer lève une exception
Python **nommée et spécifique**, jamais un simple booléen, une valeur
`None` ou un texte libre. C'est cette taxonomie qui donne un sens
précis aux mots "rejeté" et "erreur attendue" employés plus loin dans
ce plan.

| Exception | Levée par | Déclencheurs |
|---|---|---|
| `InvalidPointSetIdError(ValueError)` | `triangulator.validation.validate_point_set_id` | `pointSetId` syntaxiquement non conforme au format UUID (§5) |
| `BinaryFormatError(ValueError)` | `triangulator.binary.decode_point_set`, `decode_triangles` | payload `PointSet`/`Triangles` tronqué, trop long, ou incohérent avec son en-tête (§4) |
| `TriangulationError(Exception)` | `triangulator.geometry.triangulate` | classe de base, jamais levée directement |
| `InsufficientPointsError(TriangulationError)` | `triangulator.geometry.triangulate` | moins de 3 points en entrée (§6) |
| `DegenerateInputError(TriangulationError)` | `triangulator.geometry.triangulate` | tous les points colinéaires (§6) |

La route Flask (couche API, §7) est le seul endroit qui traduit ces
exceptions en couple (code HTTP, `code` JSON) via le mapping du §2.2 —
les fonctions internes (`validation`, `binary`, `geometry`) ne
connaissent rien de HTTP et ne font que lever ces exceptions.

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
| Intégration / système | Hors périmètre (voir §12) — couvert uniquement par le niveau "API / composant" ci-dessus | — |
| Acceptation | Critères formulés du point de vue du besoin | recoupe les niveaux ci-dessus |

## 4. Test Unitaires - Format binaire

Convention de notation : un payload est décrit comme la concaténation
de ses champs, chacun noté `<valeur:type>` (ex. `<0:u32>` pour un
entier non signé sur 4 bytes à 0, `<1.0:f32>` pour un float sur 4
bytes). L'ordre des bytes retenu est le big-endian (§2.3,
`triangulator/binary.py`).

### 4.1 Décodage de `PointSet`

Cas nominaux — entrée exacte → sortie exacte :

| # | Payload (entrée) | Points décodés (sortie attendue) |
|---|---|---|
| 1 | `<0:u32>` | `[]` |
| 2 | `<1:u32><1.5:f32><-2.5:f32>` | `[(1.5, -2.5)]` |
| 3 | `<2:u32><0.0:f32><0.0:f32><-3.5:f32><4.25:f32>` | `[(0.0, 0.0), (-3.5, 4.25)]` |
| 4 | `<1:u32><3.4e38:f32><-3.4e38:f32>` | `[(3.4e38, -3.4e38)]` (limites hautes d'un float 32 bits) |

Cas d'erreur — entrée exacte → exception attendue (`BinaryFormatError`,
voir §2.4) :

| # | Payload (entrée) | Cause |
|---|---|---|
| 5 | `b""` (0 byte) | payload vide, pas d'en-tête |
| 6 | `b"\x00\x00"` (2 bytes) | plus court que l'en-tête (4 bytes) |
| 7 | `<2:u32><1.0:f32><2.0:f32>` (12 bytes, 1,5 point) | en-tête annonce 2 points, seul 1,5 point fourni |
| 8 | `<1:u32><1.0:f32><2.0:f32><0:u32>` (16 bytes) | en-tête annonce 1 point mais 4 bytes surnuméraires après |

Le cas 8 tranche explicitement l'ancien point ouvert du §14 : un
payload plus long que ce qu'annonce l'en-tête est **rejeté**, pas
tronqué silencieusement.

### 4.2 Encodage de `PointSet`

| # | Points (entrée) | Payload attendu (sortie) |
|---|---|---|
| 1 | `[]` | `<0:u32>` (4 bytes) |
| 2 | `[(1.0, 2.0)]` | `<1:u32><1.0:f32><2.0:f32>` (12 bytes) |

Propriétés vérifiées en complément des cas ci-dessus :
- round-trip : `decode(encode(points)) == points`, sur les jeux de
  points du tableau ci-dessus et sur un jeu à N ≥ 3 points
- taille produite exactement `4 + 8*N` bytes, jamais plus, jamais moins

### 4.3 Décodage de `Triangles`

On note les sommets comme en §4.1 ; la section triangles est notée de
la même façon avec des `u32` pour les indices.

Cas nominaux :

| # | Payload (entrée) | Sortie attendue |
|---|---|---|
| 1 | sommets `[(0,0), (1,0), (0,1)]` + `<0:u32>` (0 triangle) | `(vertices=[(0,0),(1,0),(0,1)], triangles=[])` |
| 2 | sommets `[(0,0), (1,0), (0,1)]` + `<1:u32><0:u32><1:u32><2:u32>` | `(vertices=[(0,0),(1,0),(0,1)], triangles=[(0,1,2)])` |

Cas d'erreur (`BinaryFormatError`) :

| # | Payload (entrée) | Cause |
|---|---|---|
| 3 | sommets `[(0,0), (1,0), (0,1)]` + `<1:u32><0:u32><1:u32><5:u32>` | le 3e indice (5) n'existe pas dans les 3 sommets fournis |
| 4 | sommets `[(0,0), (1,0), (0,1)]` + `<2:u32><0:u32><1:u32><2:u32>` (1 triangle fourni sur 2 annoncés) | section triangles tronquée |
| 5 | section sommets elle-même invalide (ex. `b"\x00\x00"`) | l'erreur de décodage des sommets est propagée avant même de lire les triangles |

### 4.4 Encodage de `Triangles`

| # | Entrée | Payload attendu (sortie) |
|---|---|---|
| 1 | `vertices=[(1.0, 2.0)]`, `triangles=[]` | `<1:u32><1.0:f32><2.0:f32><0:u32>` (12 bytes) |

Propriétés vérifiées en complément :
- round-trip `decode(encode(vertices, triangles)) == (vertices, triangles)`
- taille produite exactement `(4 + 8*N) + (4 + 12*T)` bytes, pour N
  sommets et T triangles
- la section sommets précède toujours la section triangles, sans octet
  intercalé

## 5. Test Unitaires - Validation du pointSetId

Fonction pure, aucun double nécessaire.

**Ce que signifie "rejeté"** (précision demandée par le professeur) :
la fonction `validate_point_set_id(raw: str) -> None` lève
`InvalidPointSetIdError` (§2.4) si `raw` n'est pas un UUID
syntaxiquement valide ; elle ne retourne rien (ne lève rien) si `raw`
est valide. Pas de retour booléen, pas de `None` silencieux : l'appelant
(la route Flask) sait qu'une validation a échoué uniquement par
l'exception.

| # | Entrée (`raw`) | Comportement attendu |
|---|---|---|
| 1 | `"123e4567-e89b-12d3-a456-426614174000"` | accepté (aucune exception) |
| 2 | `""` | `InvalidPointSetIdError` |
| 3 | `"abc123"` | `InvalidPointSetIdError` |
| 4 | `"123e4567-e89b-12d3-a456-42661417400"` (35 caractères, un de moins) | `InvalidPointSetIdError` |
| 5 | `"123e4567e89b12d3a456426614174000"` (tirets absents) | `InvalidPointSetIdError` |
| 6 | `"123E4567-E89B-12D3-A456-426614174000"` (majuscules) | **à trancher** (§14) : si accepté, le documenter comme assoupli par rapport au format canonique ; si rejeté, le documenter comme un choix strict |

Le cas `None`/type inattendu est explicitement **hors contrat** : la
route Flask garantit que `pointSetId` est toujours une chaîne (extrait
d'un chemin d'URL), donc `validate_point_set_id` n'a pas à gérer un
appelant qui violerait sa signature — un `TypeError` natif de Python
en cas de mauvais usage est acceptable et n'a pas besoin d'un test
dédié.

## 6. Test Unitaires - Algo de triangularisation

**Ce que signifie "erreur attendue"** (précision demandée par le
professeur) : `triangulate(points: list[Point]) -> list[Triangle]`,
définie dans `triangulator.geometry`, lève l'une des sous-classes de
`TriangulationError` fixées en §2.4 — jamais un résultat vide ou un
`None` silencieux.

Cas nominaux — entrée exacte → sortie exacte ou propriété vérifiée :

| # | Points (entrée) | Sortie attendue |
|---|---|---|
| 1 | `[(0,0), (1,0), (0,1)]` (3 points, cas trivial) | `[(0,1,2)]` à permutation d'indices près — un seul triangle, couvrant les 3 points, d'aire non nulle |
| 2 | `[(0,0), (1,0), (1,1), (0,1)]` (carré, 4 points) | 2 triangles, chacun d'aire non nulle, dont l'union des indices couvre les 4 sommets |

Pour le cas 2 et au-delà (N ≥ 4), le résultat exact dépend de
l'algorithme retenu : plutôt que de figer une sortie littérale, les
tests vérifient les propriétés générales suivantes, déjà identifiées :
- tous les indices référencés dans les triangles existent dans le
  `PointSet` d'entrée
- chaque point d'entrée est utilisé dans au moins un triangle
- aucun triangle n'a une aire nulle (pas de 3 sommets colinéaires au
  sein d'un même triangle — y compris quand ce sous-ensemble de 3
  points est colinéaire alors que l'ensemble complet ne l'est pas)

Cas limites / erreurs :

| # | Points (entrée) | Exception attendue |
|---|---|---|
| 3 | `[]` | `InsufficientPointsError` |
| 4 | `[(0,0)]` | `InsufficientPointsError` |
| 5 | `[(0,0), (1,0)]` | `InsufficientPointsError` |
| 6 | `[(0,0), (1,0), (2,0)]` (3 points colinéaires) | `DegenerateInputError` |

Point laissé ouvert (§14), à trancher avant l'implémentation : le
comportement face à des points dupliqués (mêmes coordonnées) n'est pas
encore fixé — ignorer les doublons silencieusement, ou lever une
exception dédiée ? Les deux sont défendables et seront documentés une
fois choisis, avec leur propre ligne dans ce tableau.

## 7. Test API/composant

Utilisation du client de test Flask ; `PointSetManager` simulé via un
double injecté (mock/stub) sur le client HTTP interne.

Ces tests valident le comportement du `Triangulator` **isolé**, en
supposant que `PointSetManager` respecte sa spec. Ce ne sont pas des
tests d'intégration au sens strict (voir §12) : ils ne font intervenir
aucun vrai `PointSetManager`.

| # | Scénario | `PointSetManager` simulé | Réponse attendue |
|---|---|---|---|
| 1 | `pointSetId` mal formé | non sollicité | `400` / `INVALID_POINT_SET_ID` |
| 2 | UUID valide, ID inconnu | `404` | `404` / `NOT_FOUND` |
| 3 | UUID valide, service indisponible | `503` / timeout | `503` / `POINT_SET_MANAGER_UNAVAILABLE` |
| 4 | UUID valide, `PointSet` valide et triangulable | `200` + payload valide | `200` + `Triangles` corrects |
| 5 | UUID valide, `PointSet` insuffisant (< 3 points) | `200` + payload valide | `500` / `TRIANGULATION_FAILED` |
| 6 | UUID valide, `PointSet` colinéaire | `200` + payload valide | `500` / `TRIANGULATION_FAILED` |
| 7 | UUID valide, payload binaire corrompu renvoyé | `200` + payload invalide | `500` / `INVALID_POINT_SET_DATA` |

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

- **Seuil cible : 90 % de lignes et de branches** sur le code
  applicatif (`triangulator/`, hors configuration/lancement), mesuré
  par `coverage run --branch -m pytest` (`make coverage`). Un seuil à
  100 % est écarté : au-delà de 90 %, le coût marginal d'un test
  supplémentaire dépasse souvent sa valeur diagnostique (ex. une
  branche défensive improbable). En-dessous de 90 %, on considère que
  des cas réels du plan (§4 à §7) ne sont pas couverts.
- Une ligne ou une branche non couverte est systématiquement examinée
  avant d'être ignorée : soit elle révèle un cas de test manquant
  (à ajouter), soit elle révèle une complexité ou un chemin de code
  inutile (candidat à suppression/refactor) — jamais une raison de
  simplement ajouter un test "creux" pour faire monter le chiffre.
- La mesure de branches (`--branch`) est particulièrement utile sur le
  mapping d'erreurs du §2.2/§7 (chaque branche HTTP) et sur les cas
  limites de décodage du §4 (chaque condition de rejet).

## 10. Qualimétrie

**Précision d'intention** (suite au retour du professeur) : la
qualimétrie n'est pas un "test" au sens de ce plan — elle ne vérifie
pas un comportement métier et ne peut donc pas être écrite "avant" le
code à la façon d'un test-first. Ce qui *est* décidé avant toute
ligne de code, en revanche, ce sont les **règles elles-mêmes** : c'est
l'objet de cette section. Leur application (lancer `ruff`/`pdoc3`) est
continue, au fil de l'écriture du code, et non un contrôle a
posteriori.

Règles fixées dès maintenant :

- **Lint** : `ruff check` doit passer sans diagnostic sur l'ensemble du
  code applicatif et des tests (`make lint`). Les règles de base
  (`D`, `E`, `F`, `UP`, `B`, `SIM`, `I`) sont celles du `pyproject.toml`
  fourni ; possibilité d'en ajouter, jamais d'en retirer.
- **Documentation** : toutes les fonctions/classes publiques doivent
  avoir une docstring conforme aux règles `D` de `ruff`
  (`ruff check --select D`), avec au minimum : description, `Args`,
  `Returns`, `Raises` le cas échéant — en particulier pour les
  fonctions de décodage/encodage binaire et l'algorithme de
  triangulation, où les préconditions/erreurs (§2.4) doivent être
  listées explicitement dans `Raises`.
- **Génération de la doc** : `pdoc3` (`make doc`) doit s'exécuter sans
  erreur et produire une documentation HTML exploitable.
- **Formatage** : `ruff format --check` peut être utilisé en
  complément, sans être une exigence bloquante de l'énoncé.

Ce contrôle est vérifié à chaque exécution de `make lint` et `make doc`
tout au long de l'implémentation (pas seulement à la fin), pour éviter
une dette de qualité qui s'accumule.

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

**Périmètre confirmé** : seul `Triangulator` est implémenté dans ce
TP. `PointSetManager` n'existe pas et ne sera pas développé, même sous
une forme minimale — ni pour un test d'intégration, ni pour un test
système bout-en-bout.

Le contrat `point_set_manager.yml` fait foi, et les tests de composant
du §7 (avec double conforme à ce contrat) constituent la seule
couverture possible de cet axe dans le périmètre du TP : aucune
intégration réelle, ni test système complet
(`Client` → `PointSetManager` → `Triangulator`) n'est prévu.

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

- Comportement exact en cas de payload `PointSet` plus long que prévu :
  **tranché en §4.1/§4.2 — rejeté** (voir aussi `triangulator/binary.py`,
  qui documente le choix d'endianness big-endian).
- Comportement en cas de points dupliqués dans l'algorithme
- Sensibilité à la casse du `pointSetId`
- Valeur exacte des budgets de performance

Ces points seront documentés au fil de l'eau et repris dans `RETEX.md`.