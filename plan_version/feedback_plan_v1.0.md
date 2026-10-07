# Retours sur le Plan de Test

## Remarques Générales
* Il n'est pas nécessaire, et même **incorrect**, de complexifier le plan avec des spécificités d'implémentation de l'algorithme ou de la suite de tests (e.g. `mock.assert_not_called()`).
* **3. Qualimétrie :** Ce point manque de clarté quant à vos intentions. Idéalement, vos tests doivent être définis en grande partie **avant** de coder l'algorithme de triangulation.

---

## Retours par Point

### 3. Intégration / Système
* On cherche à implémenter uniquement `Triangulator`.

### 4. Test Format Binaire
* Il manque la spécification des **sorties attendues pour chaque entrée** (e.g., coordonnées négatives, nulles, etc.). 
* *Question :* Quelles sont vos entrées exactes et le résultat attendu pour chaque test ?

### 5. Tests Validation PointSetId
* Que signifie concrètement le terme **"rejeté"** ? (Exception levée, code d'erreur, rejet silencieux, etc.)

### 6. Tests Triangulation
* Vous mentionnez une **"erreur attendue"**, mais que allez-vous vérifier exactement dans votre test ? 
* Normalement, vous devriez spécifier le code d'erreur obtenu ainsi que sa provenance (quel module le retourne).

---

## Couverture de Code
* **Définir un seuil à atteindre.** Les lignes non couvertes indiquent souvent une complexité du code trop élevée, d'où tout l'intérêt de la mesurer.