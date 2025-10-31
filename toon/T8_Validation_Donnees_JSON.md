---
id: T8
name: "T8 — Schémas JSON & linter des données"
kind: ticket
status: todo
epic: E5
owners:
  - engineering
  - qa
estimate: 5j
risks:
  - Schéma trop strict bloquant les itérations
  - Incohérence entre schéma et runtime
  - Temps d'exécution trop long en CI
---

# T8 — Schémas JSON & linter des données

## Résumé
Créer des schémas JSON et un outil de validation pour sécuriser les données items/affixes/chunks/loot tables et empêcher les régressions de contenu.

## Pré-requis exacts
- Inventaire complet des fichiers JSON à valider.
- Choisir `jsonschema` (Python) comme validator.
- Définir conventions de nommage (snake_case, ids uniques).

## Tâches détaillées
1. **Schémas**
   - Rédiger `schemas/items.schema.json`, `schemas/affixes.schema.json`, `schemas/chunks.schema.json`, `schemas/loot_tables.schema.json`.
   - Documenter les champs obligatoires, types, bornes.
2. **CLI de validation**
   - Créer `tools/validate_data.py` qui parcourt `data/` et valide chaque fichier selon son schéma.
   - Option `--watch` pour usage local (reload automatique).
3. **Intégration CI**
   - Ajouter un job `data-lint` exécutant `python tools/validate_data.py --ci`.
   - Rendre le job bloquant sur PR.
4. **Jeu de tests**
   - Générer 1 000 items aléatoires avec `tools/generate_dummy_items.py` et valider qu'aucune erreur ne survient.
   - Ajouter un fixture JSON invalide pour assurer que le linter échoue correctement.

## Tests & validations obligatoires
- `python tools/validate_data.py`
- `python tools/generate_dummy_items.py --count 1000`
- Vérifier que le job CI échoue volontairement sur le fixture invalide.

## Definition of Done
- Tous les JSON existants conformes aux schémas.
- CI bloque toute PR avec données invalides.
- Documentation `docs/data/validation.md` mise à jour.

## Livrables / Artefacts
- `schemas/*.schema.json`
- `tools/validate_data.py`
- `tools/generate_dummy_items.py`
- `docs/data/validation.md`
