---
id: T4
name: "T4 — Versioning & migration des sauvegardes"
kind: ticket
status: todo
epic: E6
owners:
  - engineering
  - qa
estimate: 8j
risks:
  - Corruption silencieuse lors d'une migration partielle
  - Explosion de la taille disque à cause des backups
  - Perte de compatibilité rétroactive au-delà de deux versions
  - Paramètres KBM non migrés qui forcent un retour aux presets manette
---

# T4 — Versioning & migration des sauvegardes

## Résumé
Sécuriser l'évolution des sauvegardes en introduisant un champ de version, des migrateurs incrémentaux testés, des backups atomiques et un plan de réparation utilisateur pour éviter la perte de progression.

## Pré-requis exacts
- Exporter la configuration des contrôles KBM (T1) pour garantir la migration des options input sans régression.
- Définir les formats de sauvegarde actuels (`meta`, `run`, `profile`).
- Choisir un format de checksum (SHA256) et bibliothèque (Godot `Crypto`).
- Budget disque pour les backups (≥3× taille moyenne de save).

## Tâches détaillées
1. **Versioning**
   - Ajouter `save_version` dans chaque fichier `*.save`.
   - Mettre à jour `GameConfig` pour écrire la version courante (ex: `3`).
2. **Migrateurs incrémentaux**
   - Créer `scripts/save/migrations/` avec un fichier par version `migration_vX_to_vY.gd`.
   - Implémenter un orchestrateur `SaveMigrator` qui applique chaque step en séquence.
   - Couvrir les cas de suppression de champs, renommage, ajout.
3. **Backups & intégrité**
   - Avant migration, copier la save vers `user://backup/<timestamp>/`.
   - Calculer checksum avant/après, stocker dans `user://backup/checksums.json`.
4. **Plan de restauration**
   - Ajouter un écran `scenes/ui/save_repair_dialog.tscn` permettant de restaurer un backup.
   - Documenter la procédure pour le support utilisateur.
5. **Tests automatisés**
   - Générer 100 sauvegardes synthétiques (v1→v3) via `tools/generate_fake_saves.py`.
   - Lancer la migration et vérifier 0 erreur.

## Tests & validations obligatoires
- `gdunit4 -s tests/save/test_save_migrations.gd`
- `python tools/generate_fake_saves.py --from 1 --to 3 --count 100`
- `godot --headless --path ./game --run test_save_repair_dialog`

## Definition of Done
- Migrations v1→v3 réussies sur 100 saves sans corruption.
- Backup restaurable via l'écran dédié.
- Checksum cohérent entre version originale et migrée.
- Documentation `docs/saves/migration_plan.md` mise à jour.

## Livrables / Artefacts
- `scripts/save/SaveMigrator.gd`
- `scripts/save/migrations/`
- `scenes/ui/save_repair_dialog.tscn`
- `tools/generate_fake_saves.py`
- `docs/saves/migration_plan.md`
