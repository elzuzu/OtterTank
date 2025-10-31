---
id: E8
name: "EPIC E8 — Endgame & Opérations"
kind: epic
status: todo
owners:
  - engineering
summary: "Construire les modes endgame (Opérations à clés, Rifts, Warfront hebdomadaire), leur scaling par World Tier et les récompenses associées."
dependencies:
  - E1
  - E2
  - E3
  - E4
  - E5
  - E6
  - E7
critical_path: true
---

# EPIC E8 — Endgame & Opérations

## Objectif pédagogique
Tu dois créer les modes rejouables : Opérations à clé, Rifts à menace croissante et Warfront hebdo. Chaque sprint précise les données, scripts et tests.

## Préparation commune
- Crée `docs/endgame/endgame_design.md` (résume objectifs, durées, récompenses).
- Ajoute `game/data/world_tiers.json` avec base des niveaux (`tier`, `enemy_hp_multiplier`, `loot_ilvl_range`).
- Prépare un dossier `docs/endgame/screenshots/` pour les captures.

## Sprint 1 — World Tiers & scaling

### Résultat attendu
Un système lit `world_tiers.json`, applique multiplicateurs à l'IA et aux loots, et expose un sélecteur de tier.

### Checklist
1. **Data**
   - Exemple `world_tiers.json` :
     ```json
     {
       "tiers": [
         {"id": 1, "enemy_hp_multiplier": 1.0, "loot_ilvl_min": 10, "loot_ilvl_max": 20},
         {"id": 2, "enemy_hp_multiplier": 1.2, "loot_ilvl_min": 21, "loot_ilvl_max": 30}
       ]
     }
     ```
2. **WorldTierManager**
   - `scripts/endgame/WorldTierManager.gd` : charge JSON, expose `set_tier(id)`, `get_current()`, `apply_to_enemy(enemy)`, `get_loot_range()`.
   - `EnemyGrunt._ready()` appelle `WorldTierManager.apply_to_enemy(self)` (multiplie HP).
3. **UI**
   - `scenes/ui/WorldTierSelector.tscn` : `PopupPanel` avec boutons `Tier 1..5`. Quand joueur change, sauvegarde dans `MetaState`.
4. **Tests**
   - `test_worldtier_hp_multiplier.gd` : tier 2 -> HP = base * 1.2.
   - `test_worldtier_loot_range.gd` : range correspond.

### Critères d'acceptation
- Tier sélectionné persiste entre runs (sauvegardé via E6).
- UI affiche un lock si tech tree pas débloqué (`TechTreeManager`).

## Sprint 2 — Opérations à clés

### Résultat attendu
Des missions courtes générées via `LevelGenerator` mais modifiées par des "Key Mods". Les clés sont consommées à l'entrée.

### Checklist
1. **Key Items**
   - Ajoute `items.json` : `key_operation_basic` (slot `key`, non stackable).
2. **KeyInventory**
   - `scripts/endgame/KeyInventory.gd` : gère nombre de clés par type, expose `consume(key_id)`.
   - Connecte `InventoryBus` pour ajouter clés via drops.
3. **OperationConfig**
   - `data/operations.json` :
     ```json
     {
       "operations": [
         {"id": "steel_rush", "biome": "urban", "duration": 600, "modifiers": ["steel_floor", "elite_snipers"], "rewards": {"keys": {"key_operation_basic": 1}, "loot": 2}}
       ]
     }
     ```
4. **OperationRunner**
   - `scripts/endgame/OperationRunner.gd` :
     ```gdscript
     extends Node

     class_name OperationRunner

     func start_operation(id: String, key_id: String) -> void:
         if not KeyInventory.consume(key_id):
             push_error("Missing key")
             return
         var config := _find_config(id)
         LevelGenerator.generate(_build_blueprint(config))
         DirectorController.force_modifier(config["modifiers"])
         RunAnalytics.start_run()
     ```
   - `_build_blueprint` modifie `LevelBlueprint` (duration = 10 min, modifs appliqués).
5. **UI**
   - `scenes/ui/OperationBoard.tscn` : liste d'opérations disponibles + bouton `Start`. Affiche les modificateurs et les récompenses.

### Tests & validations
- `test_key_inventory_consume.gd` : consommer une clé la décrémente.
- `test_operation_requires_key.gd` : start sans clé → erreur.
- Capture `docs/endgame/screenshots/operation_board.png`.

### Critères d'acceptation
- Consommation de clé loggée via `Telemetry.increment("keys_spent")`.
- OperationRunner ne laisse pas la RunAnalytics non terminée (appelle `RunAnalytics.end_run`).

## Sprint 3 — Rifts & Threat scaling

### Résultat attendu
Un mode Rift à durée 10 minutes avec menaces croissantes, s'appuyant sur la Threat Clock (E7) et offrant récompenses selon palier.

### Checklist
1. **RiftConfig**
   - `data/rifts.json` : `id`, `threat_rate`, `rewards` (per palier 2, 4, 6, 8, 10 minutes).
2. **RiftRunner**
   - `scripts/endgame/RiftRunner.gd` :
     ```gdscript
     extends Node

     class_name RiftRunner

     @export var threat_clock: ThreatClock

     var _elapsed := 0.0

     func start_rift(config_id: String) -> void:
         _elapsed = 0.0
         var config := _load_config(config_id)
         threat_clock.add(config["threat_rate"]) # initial push
         RunAnalytics.start_run()

     func _process(delta: float) -> void:
         _elapsed += delta
         if int(_elapsed) % 60 == 0:
             threat_clock.add(5.0)
         if _elapsed >= 600.0:
             _complete_rift()

     func _complete_rift() -> void:
         RunAnalytics.end_run("success")
         _grant_rewards()
     ```
3. **Rewards**
   - `RiftRunner._grant_rewards()` lit config et accorde loot (via `DropManager.force_drop`).
4. **UI**
   - `scenes/ui/RiftTimer.tscn` affiche le temps restant, la menace actuelle, et les paliers atteints.
5. **Tests**
   - `test_rift_timer_duration.gd` : simule 600 s → `_complete_rift` appelé.
   - `test_rift_rewards_distribution.gd` : s'assurer que chaque palier donne la bonne récompense.

### Critères d'acceptation
- Le joueur peut abandonner (`Esc`) → `RunAnalytics.end_run("abandon")`.
- Threat rate augmente de 5 toutes les minutes (log `Telemetry.increment("rift_threat_tick")`).

## Sprint 4 — Warfront hebdomadaire & matchmaking

### Résultat attendu
Un Warfront global (atlas E6) avec rotation hebdomadaire, modificateurs communs et "Nemesis Tank" asynchrone.

### Checklist
1. **WarfrontSchedule**
   - `scripts/endgame/WarfrontScheduler.gd` : calcule la semaine courante (`Time.get_datetime_string_from_system`). Associe modificateur global (ex: `mud_everywhere`).
2. **Nemesis Tank**
   - `scripts/endgame/NemesisManager.gd` : télécharge (fake) un build JSON depuis `res://data/nemesis_samples.json`, instancie un boss reprenant l'équipement.
   - Lorsque le joueur termine une run, sauvegarde son build simplifié dans `user://ghosts/last_build.json`.
3. **Matchmaking UI**
   - `scenes/ui/WarfrontPanel.tscn` : affiche les modificateurs hebdo, les récompenses, bouton `Fight Nemesis`.
4. **Integration**
   - Quand le joueur lance un Warfront, appelle `LevelGenerator` avec modif globale, `DirectorController.force_modifier` (E7) pour adapter.
5. **Tests**
   - `test_warfront_schedule_rotates.gd` : change la date → modificateur change.
   - `test_nemesis_spawn_equipment.gd` : assure que le boss possède les modules listés.
6. **Documentation**
   - Ajoute `docs/endgame/endgame_design.md` section "Warfront" (table modifs vs récompenses).

### Critères d'acceptation
- Warfront se met à jour toutes les 168 h (7 jours).
- Nemesis drop garanti un légendaire (utilise `DropManager.force_drop`).

## Dépendances
- Requiert Director (E7), Meta (E6) et Loot (E5).
- Fournit contenus aux prochaines saisons (E9 peut s'y brancher).

## Risques & mitigations
- **Complexité** : garde un diagramme de flux `docs/endgame/flowchart.png`.
- **Surcharge Reward** : limite à 3 loot par palier (config JSON). Ajoute tests pour les limites.
- **Matchmaking fake** : clarifie dans README que c'est asynchrone (pas de réseau).
