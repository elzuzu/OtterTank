---
id: E5
name: "EPIC E5 — Loot & Itemisation"
kind: epic
status: todo
owners:
  - engineering
summary: "Mettre en place le système de loot (tables, raretés, drops), les modules Field Kit, l'inventaire Cargo Tetris et les effets sur les ressources."
dependencies:
  - E1
  - E2
  - E3
  - E4
critical_path: true
---

# EPIC E5 — Loot & Itemisation

## Objectif pédagogique
Tu dois créer un système de loot inspiré d'ARPG mais simplifié, avec modules de tank modulaires, drop tables adaptatives et gestion d'inventaire Tetris. Suis les recettes mot pour mot.

## Préparation commune
- Vérifie les contraintes d'équipement de T1 : aucun module ne doit introduire de visée assistée ou de smoothing implicite.
- Ajoute `docs/data/loot_tables.csv` (colonnes : `DropTableID`, `ItemID`, `Weight`, `WorldTier`, `Notes`).
- Télécharge les icônes Kenney `Loot Pack` pour les items (`game/assets/icons/`).
- Active `Project Settings > AutoLoad > res://scripts/inventory/InventoryBus.gd` (sera créé dans Sprint 2).

## Sprint 1 — Base de données d'items

### Résultat attendu
Une base d'items en JSON (`res://data/items.json`) chargée par un `ItemDatabase` fournissant stats et effets.

### Checklist
1. **Définir le format JSON**
   - Crée `game/data/items.json` avec ce squelette :
     ```json
     {
       "items": [
         {
           "id": "turret_basic",
           "slot": "turret",
           "rarity": "common",
           "base_stats": {"damage": 10, "heat_gen": 3},
           "affixes": [{"type": "projectile_count", "value": 1}],
           "resource_modifiers": {"ammo_cost": 1}
         }
       ]
     }
     ```
   - Ajoute au moins 20 entrées couvrant les slots `chassis`, `turret`, `tracks`, `engine`, `utility`.
2. **ItemDatabase**
   - `scripts/loot/ItemDatabase.gd` :
     ```gdscript
     extends Node

     class_name ItemDatabase

     var _items: Dictionary = {}

     func load_from_file(path: String) -> void:
         var file := FileAccess.open(path, FileAccess.READ)
         var data := JSON.parse_string(file.get_as_text())
         for item in data["items"]:
             _items[item["id"]] = item

     func get_item(id: String) -> Dictionary:
         return _items.get(id, {})

     func get_items_by_slot(slot: String) -> Array:
         return _items.values().filter(func(i): return i["slot"] == slot)
     ```
   - Appelle `load_from_file("res://data/items.json")` dans `_ready()`.
3. **Tests**
   - `test_item_database_loads.gd` : vérifie que 20 items sont chargés.
   - `test_item_by_slot.gd` : assure que `get_items_by_slot("turret")` renvoie >0.
4. **Documentation**
   - Complète `docs/data/loot_tables.csv` avec au moins 5 tables (`story_act1`, `mini_boss`, `world_tier_1`, etc.).

### Critères d'acceptation
- JSON valide (utilise `jq . game/data/items.json` pour vérifier).
- Tous les items ont un champ `rarity` parmi `common`, `rare`, `epic`, `legendary`.

## Sprint 2 — Loot tables & drops

### Résultat attendu
Des ennemis laissent tomber des items selon leur table de loot, avec animation de drop et ramassage interactif.

### Checklist
1. **InventoryBus**
   - `scripts/inventory/InventoryBus.gd` :
     ```gdscript
     extends Node

     signal item_picked_up(item_id: String)

     func notify_pickup(item_id: String) -> void:
         emit_signal("item_picked_up", item_id)
     ```
   - Ajoute en autoload.
2. **LootTable**
   - `scripts/loot/LootTable.gd` : charge `docs/data/loot_tables.csv` (utilise `CSVParser`). Fournit `pick_item(table_id: String, rng: RandomNumberGenerator) -> String`.
3. **DropManager**
   - `scripts/loot/DropManager.gd` : écoute `EventBus` (`ENEMY_DIED`). Selon l'ennemi (`grunt` -> `world_tier_1`, `miniboss` -> `mini_boss`), instancie `LootPickup.tscn`.
   - `LootPickup.tscn` : `Area2D` + `Sprite2D` (icône) + `AnimationPlayer` pour pulser. Script :
     ```gdscript
     extends Area2D

     @export var item_id: String

     func _on_body_entered(body: Node) -> void:
         if body.is_in_group("player"):
             InventoryBus.notify_pickup(item_id)
             queue_free()
     ```
   - Ajoute un label flottant `+ItemName` via `Tween`.
4. **Feedback**
   - Sons : télécharge `pickupCoin.wav` (Kenney) → `AudioStreamPlayer2D`.
   - Ajoute `EventBus.emit_event("LOOT_PICKED", {"item": item_id})`.

### Tests & validations
- Tue 10 ennemis → capture un screenshot montrant des loot au sol (`docs/previews/sprint2_loot.png`).
- Tests : `test_loot_table_pick.gd` (probabilité). Simule 1000 picks et vérifie que items rares tombent < 10%.
- `test_drop_manager_emits_signal.gd` : mock EventBus, s'assure que `LOOT_PICKED` est émis.

### Critères d'acceptation
- Aucun loot ne reste inaccessible (rayon de pickup >= 24 px).
- Les icônes correspondent à l'item (vérifier via `item_id`).

## Sprint 3 — Field Kit modulaire & Cargo Tetris

### Résultat attendu
Inventaire Tetris 6x4 permettant d'équiper modules à chaud via bulle de pit-stop, avec UI radiale.

### Checklist
1. **Inventaire grille**
   - `scripts/inventory/CargoGrid.gd` : gère une matrice 6 colonnes x 4 lignes.
     ```gdscript
     extends Resource

     class_name CargoGrid

     var cells := []

     func _init():
         cells.resize(6)
         for i in range(6):
             cells[i] = []
             cells[i].resize(4)
             for j in range(4):
                 cells[i][j] = null

     func can_place(item_shape: Array[Vector2i], origin: Vector2i) -> bool:
         for offset in item_shape:
             var pos := origin + offset
             if pos.x < 0 or pos.y < 0 or pos.x >= 6 or pos.y >= 4:
                 return false
             if cells[pos.x][pos.y] != null:
                 return false
         return true

     func place(item_id: String, item_shape: Array[Vector2i], origin: Vector2i) -> bool:
         if not can_place(item_shape, origin):
             return false
         for offset in item_shape:
             var pos := origin + offset
             cells[pos.x][pos.y] = item_id
         return true
     ```
   - Ajoute un test `test_cargo_grid_place.gd`.
2. **UI inventaire**
   - `scenes/ui/CargoPanel.tscn` : `Control` + `GridContainer` 6x4. Chaque case `TextureRect` change de couleur si occupée.
   - `scripts/ui/CargoPanel.gd` : dessine la forme de l'item lors du drag.
3. **Pit-stop bubble**
   - `scenes/core/PitStopBubble.tscn` : `Area2D` qui, quand le tank entre, fige le temps (`Engine.time_scale = 0.3`), affiche la `CargoPanel` et autorise le swap.
   - Ajoute cooldown 60 s (`Timer`).
4. **Radial menu**
   - `scenes/ui/HotSwapRadial.tscn` : `CanvasLayer` avec 5 segments (`chassis`, `turret`, `tracks`, `engine`, `utility`).
   - `scripts/ui/HotSwapRadial.gd` : quand le joueur survole un segment, affiche les items disponibles, confirme avec `E`.
5. **Equipement**
   - `scripts/inventory/EquipmentManager.gd` : gère les slots et applique les stats au tank (`tank.move_speed += item["base_stats"]["move_speed"]`).
   - Connecte `InventoryBus.item_picked_up` → `CargoGrid.place` → `EquipmentManager`.

### Tests & validations
- Capture un GIF `docs/previews/sprint3_inventory.gif` montrant un swap.
- Tests : `test_equipment_applies_stats.gd` (mock item), `test_pit_stop_cooldown.gd` (vérifie temps).
- QA : s'assurer que quitter la bulle remet `Engine.time_scale = 1`.

### Critères d'acceptation
- Aucun item ne disparaît lors du drag (utilise `set_process_unhandled_input(true)`).
- Radial menu navigable au clavier (`left/right/up/down`).

## Sprint 4 — Effets sur ressources & légendaires

### Résultat attendu
Les items modifient Fuel/Heat/Ammo selon leurs affixes, introduction de 3 légendaires avec twists uniques.

### Checklist
1. **Affixes**
   - Implémente dans `EquipmentManager` la lecture de `affixes` : par exemple `{"type": "fuel_regen", "value": 1.5}` → appelle `ResourceManager.regenerate_fuel` passivement.
   - Ajoute un dictionnaire `AFFIX_HANDLERS` mappant type → Callable.
2. **Légendaires**
   - Ajoute 3 items `legendary` dans `items.json` :
     - `turret_rail_overpenetration` (projectiles traversent 2 murs, +25% heat).
     - `engine_phantom_drive` (+dash, consomme 10 fuel à l'activation, `EventBus.emit_event("DASH_TRIGGERED")`).
     - `utility_drone_forge` (spawn un drone support toutes 60 s).
3. **Hot-swap feedback**
   - Lorsqu'un item est équipé, affiche un toast `Equipped: {name}` (scène `Toast.tscn`).
   - Ajoute SFX `equip.wav` (Kenney UI).
4. **Tests**
   - `test_affix_handlers_apply.gd` : vérifie que `fuel_regen` augmente Fuel au bout de 10 s (utilise `await` 600 frames).
   - `test_legendary_unique_effects.gd` : check `engine_phantom_drive` déclenche dash cooldown.
5. **Documentation**
   - Mettre à jour `docs/data/loot_tables.csv` en marquant les légendaires (colonne `Notes`).
   - Ajoute `docs/previews/legendary_cards.png` (mockup Figma).

### Critères d'acceptation
- Les affixes ne se cumulent pas deux fois (utilise `Set` pour identifier uniques).
- Tous les légendaires ont des tooltips `desc_fr` et `desc_en`.

## Dépendances
- Utilise ResourceManager (E2), Enemy events (E3), Objectives (E4) pour drop tables.
- Alimente E7 (director adaptatif) via `InventoryBus` (build du joueur).

## Risques & mitigations
- **Données corrompues** : ajouter un test `test_items_json_schema.gd` validant le schema (utilise `JSONSchemaValidator` Godot addon).
- **UI complexe** : documente chaque interaction dans `docs/ux/inventory_flow.md`.
- **Balance** : log drop rates dans `Telemetry` (`Telemetry.increment("loot_common")`).
