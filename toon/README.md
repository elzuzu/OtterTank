# Backlog TOON — OtterTank

Ce répertoire contient les tickets TOON pour piloter la réalisation complète d'OtterTank dans Godot 4.x. Chaque fichier `EPIC_Exx_*.md` décrit un epic, découpé en sprints avec étapes détaillées, tests et critères d'acceptation. Les métadonnées YAML en tête de fichier (id, dépendances, chemin critique) permettent d'orchestrer l'exécution via l'outil [TOON](https://github.com/johannschopplich/toon).

## Structure (lis la colonne "Tu dois" avant de commencer)

| Fichier | Tu dois | Notes |
| --- | --- | --- |
| `EPIC_E1_Core_Framework.md` | Suivre à la lettre l'installation Godot, scripts shell, CI. | Commence toujours par vérifier `which godot`. |
| `EPIC_E2_Tank_TriRessource.md` | Implémenter le tank, le HUD tri-ressource et les événements logistiques. | Télécharge les assets Kenney listés, sinon le sprint échoue. |
| `EPIC_E3_Horde_IA.md` | Créer les ennemis, carcasses persistantes et spawners adaptatifs. | N'avance pas si les tests gdUnit4 échouent. |
| `EPIC_E4_Procedural_Objectifs.md` | Générer les niveaux chunk-based et les objectifs. | Mets à jour `docs/procgen/*.md` après chaque sprint. |
| `EPIC_E5_Loot_Itemisation.md` | Gérer items, loot tables, inventaire Tetris. | Vérifie `jq` sur `items.json` avant commit. |
| `EPIC_E6_Meta_Sauvegarde.md` | Construire la méta progression et la sauvegarde chiffrée. | Ne jamais commiter `.env`, suis la checklist. |
| `EPIC_E7_Director_ThreatClock.md` | Programmer le director adaptatif et la Threat Clock. | Remplis le dossier `docs/director/` sinon sprint invalide. |
| `EPIC_E8_Endgame_Operations.md` | Mettre en place Opérations, Rifts et Warfront. | Chaque sprint exige captures dans `docs/endgame/`. |
| `EPIC_E9_Allies_Radio_RTS.md` | Ajouter les alliés radio RTS. | Optionnel mais doit respecter toutes les validations si tu le lances. |

## Chemin critique global

1. **E1** → 2. **E2** → 3. **E3** → 4. **E4** → 5. **E5** → 6. **E6** → 7. **E7** → 8. **E8**.
9. **E9** peut être intégré après E7/E8 pour enrichir le gameplay asymétrique.

## Utilisation

1. Installer TOON (`npm install -g @johannschopplich/toon`).
2. Exécuter `toon render` depuis la racine du repo pour voir la roadmap.
3. Avant de démarrer un sprint, lis la section "Prérequis" du fichier correspondant et prépare tous les assets listés.
4. Après chaque sprint, colle les sorties CLI demandées (captures, logs) dans le dossier `docs/` indiqué.

Chaque sprint contient :
- **Checklist** : à cocher physiquement dans ton suivi (copie/colle dans Notion ou Markdown).
- **Tests** : exécute exactement les commandes listées, capture la sortie.
- **Critères d'acceptation** : si un seul point manque, le sprint est refusé.

Si tu n'es pas sûr : recommence l'étape depuis le début, n'invente pas. Les placeholders (SFX, assets) doivent provenir des liens officiels (Kenney, Freesound, docs Godot) mentionnés dans chaque epic.
