---
id: E2
name: "EPIC E2 — Tank & Tri-Ressource"
kind: epic
status: todo
owners:
  - engineering
summary: "Implémenter le contrôle du tank, la gestion Fuel/Heat/Ammo, le HUD associé et les interactions terrain." 
dependencies:
  - E1
critical_path: true
---

# EPIC E2 — Tank & Tri-Ressource

## Objectif pédagogique
Tu dois produire un tank jouable avec trois jauges (Fuel, Heat, Ammo) qui réagissent aux inputs, aux types de terrain et à l'utilisation d'abilities. Chaque étape est une recette : suis-la sans improviser.

## Préparation commune
- Assure-toi d'avoir terminé E1 (tests gdUnit4 verts, pipeline CI OK).
- Télécharge les assets placeholder gratuits :
  - Sprite du tank : <https://kenney.nl/assets/topdown-tanks-redux> → utilise `tankBody_darkOutline.png`.
  - Tileset : <https://kenney.nl/assets/topdown-shooter> → `tile_Grass.png`, `tile_Mud.png`, `tile_Metal.png`.
- Copie les sprites dans `game/assets/kenney/`. Commande :
  ```bash
  mkdir -p game/assets/kenney && cp /chemin/vers/tankBody_darkOutline.png game/assets/kenney/
  ```
  (remplace `/chemin/vers/` par ton vrai dossier, sinon ça casse.)

## Sprint 1 — TankController & Physique basique

### Résultat attendu
Un tank (châssis + tourelle) contrôlable en twin-stick, avec déplacement, rotation et tir de projectiles dummy.

### Checklist chronologique
1. **Créer la scène Tank**
   - `res://scenes/core/Tank.tscn` :
     - `KinematicBody2D` (ou `CharacterBody2D` Godot 4) nommé `Tank`.
     - Enfant `Sprite2D` (texture `tankBody_darkOutline.png`).
     - Enfant `Node2D` `Turret` avec `Sprite2D` (utilise `tankBarrel_dark.png`).
   - Script `res://scripts/core/Tank.gd` :
     ```gdscript
     extends CharacterBody2D

     @export var move_speed: float = 220.0
     @export var rotation_speed: float = 3.0
     @export var turret_rotation_speed: float = 5.0
     @export var projectile_scene: PackedScene = preload("res://scenes/core/Projectile.tscn")

     var aim_vector: Vector2 = Vector2.RIGHT

     func _physics_process(_delta: float) -> void:
         var input_vector := Vector2.ZERO
         input_vector.x = Input.get_action_strength("move_right") - Input.get_action_strength("move_left")
         input_vector.y = Input.get_action_strength("move_down") - Input.get_action_strength("move_up")
         velocity = input_vector.normalized() * move_speed
         move_and_slide()

     func update_aim_from_input(delta: float) -> void:
         var aim_x := Input.get_action_strength("aim_right") - Input.get_action_strength("aim_left")
         var aim_y := Input.get_action_strength("aim_down") - Input.get_action_strength("aim_up")
         var new_aim := Vector2(aim_x, aim_y)
         if new_aim.length() > 0.1:
             aim_vector = new_aim.normalized()
             var target_angle := aim_vector.angle()
             $Turret.rotation = lerp_angle($Turret.rotation, target_angle, turret_rotation_speed * delta)

     func try_fire() -> void:
         if Input.is_action_just_pressed("shoot"):
             var projectile := projectile_scene.instantiate()
             projectile.global_position = $Turret.global_position + aim_vector * 32
             projectile.rotation = aim_vector.angle()
             get_tree().current_scene.add_child(projectile)
             EventBus.emit_event("PROJECTILE_FIRED", {"position": projectile.global_position})
     ```
2. **Créer `Projectile.tscn`**
   - `Area2D` avec `CollisionShape2D` (Circle radius 4) et `Sprite2D` (utilise `bulletYellow.png`).
   - Script `res://scripts/core/Projectile.gd` :
     ```gdscript
     extends Area2D

     @export var speed: float = 900.0
     var direction: Vector2 = Vector2.RIGHT

     func _ready() -> void:
         set_process(true)

     func _process(delta: float) -> void:
         position += direction * speed * delta
         if not get_viewport_rect().has_point(global_position):
             queue_free()
     ```
3. **Intégrer le tank dans `Game.tscn`**
   - Instancie `Tank.tscn` sous `Game`.
   - Dans `Game.gd`, connecte le signal `tick_started` pour appeler `tank.update_aim_from_input(delta)` et `tank.try_fire()`.
4. **Configurer InputMap** (ajoute les actions si manquantes) : `aim_left/right/up/down` pour manette, `mouse_motion` (Sprint 2).

### Tests & validations
- Dans l'éditeur, appuie sur `F5` : le tank se déplace sur un fond gris.
- Les projectiles sortent dans la direction de la tourelle.
- `Telemetry.increment("projectiles")` est appelé dans `try_fire()` (ajoute l'appel).
- Ajoute une vidéo GIF (5 s) montrant le tank qui tire (peut être un enregistrement OBS) dans le dossier `docs/previews/sprint1.gif`.

### Critères d'acceptation
- `Tank.gd` n'utilise pas de `Input.is_action_pressed` directement dans `_process` pour le tir (uniquement `try_fire`).
- Tous les nœuds sont nommés comme indiqué (`Tank`, `Turret`).

## Sprint 2 — Jauges Fuel/Heat/Ammo + HUD

### Résultat attendu
Trois jauges affichées à l'écran, avec consommation et régénération réaliste.

### Checklist
1. **Créer un gestionnaire de ressources**
   - `res://scripts/systems/ResourceManager.gd` :
     ```gdscript
     extends Node

     class_name ResourceManager

     signal fuel_changed(value: float)
     signal heat_changed(value: float)
     signal ammo_changed(value: int)

     var fuel: float = 100.0
     var heat: float = 0.0
     var ammo: int = 50

     const FUEL_MAX := 100.0
     const HEAT_MAX := 100.0
     const AMMO_MAX := 60

     func consume_fuel(amount: float) -> void:
         fuel = clamp(fuel - amount, 0.0, FUEL_MAX)
         emit_signal("fuel_changed", fuel)

     func regenerate_fuel(amount: float) -> void:
         fuel = clamp(fuel + amount, 0.0, FUEL_MAX)
         emit_signal("fuel_changed", fuel)

     func add_heat(amount: float) -> void:
         heat = clamp(heat + amount, 0.0, HEAT_MAX)
         emit_signal("heat_changed", heat)

     func dissipate_heat(amount: float) -> void:
         heat = clamp(heat - amount, 0.0, HEAT_MAX)
         emit_signal("heat_changed", heat)

     func spend_ammo(amount: int) -> bool:
         if ammo < amount:
             return false
         ammo -= amount
         emit_signal("ammo_changed", ammo)
         return true

     func refill_ammo(amount: int) -> void:
         ammo = clamp(ammo + amount, 0, AMMO_MAX)
         emit_signal("ammo_changed", ammo)
     ```
2. **Attacher le ResourceManager au Tank**
   - Ajoute `@export var resource_manager: ResourceManager` dans `Tank.gd`.
   - Dans `_physics_process`, appelle `resource_manager.consume_fuel(0.1)` quand le tank se déplace.
   - Dans `try_fire`, vérifie `if not resource_manager.spend_ammo(1): return`.
   - Ajoute `resource_manager.add_heat(5.0)` à chaque tir et `resource_manager.dissipate_heat(1.5 * delta)` dans `_physics_process`.
3. **Créer le HUD**
   - `res://scenes/ui/Hud.tscn` (CanvasLayer) → `VBoxContainer` avec trois `TextureProgressBar` (`FuelBar`, `HeatBar`, `AmmoBar`).
   - Script `res://scripts/ui/Hud.gd` :
     ```gdscript
     extends CanvasLayer

     func bind(manager: ResourceManager) -> void:
         manager.connect("fuel_changed", Callable(self, "_on_fuel_changed"))
         manager.connect("heat_changed", Callable(self, "_on_heat_changed"))
         manager.connect("ammo_changed", Callable(self, "_on_ammo_changed"))
         _on_fuel_changed(manager.fuel)
         _on_heat_changed(manager.heat)
         _on_ammo_changed(manager.ammo)

     func _on_fuel_changed(value: float) -> void:
         $VBoxContainer/FuelBar.value = value

     func _on_heat_changed(value: float) -> void:
         $VBoxContainer/HeatBar.value = value

     func _on_ammo_changed(value: int) -> void:
         $VBoxContainer/AmmoBar.value = value
     ```
   - Instancie le HUD dans `Game.tscn` et appelle `hud.bind(resource_manager)` depuis `Game.gd` (crée une instance de `ResourceManager` dans `Game` et passe la référence au Tank via exported variable).
4. **Styles**
   - Configure `FuelBar.tint_progress = #00FF88`, `HeatBar = #FF5522`, `AmmoBar = #FFD400` pour lisibilité.

### Tests & validations
- Dans le jeu, la barre Fuel diminue quand tu bouges et remonte quand tu t'arrêtes (ajoute `resource_manager.regenerate_fuel(0.15 * delta)` si Fuel < max).
- Heat augmente en tirant, se dissipe au repos.
- Ammo atteint 0 → plus de tir possible (affiche message `print("Out of ammo")`).
- Capture une capture d'écran HUD et sauvegarde `docs/previews/sprint2.png`.

### Critères d'acceptation
- Les signaux du ResourceManager sont connectés une seule fois (pas de `connect` dupliqué).
- Les textures HUD sont présentes et lisibles en 1080p.

## Sprint 3 — Interaction terrain (boue, acier) + surchauffe

### Résultat attendu
Le terrain modifie la consommation de ressources : boue refroidit mais ralenti, acier provoque ricochet sonore.

### Checklist
1. **Créer la tilemap principale**
   - `scenes/core/Level.tscn` → `Node2D` avec `TileMap` (`TerrainTileMap`).
   - Importer `tilemap_tileset.tres` en utilisant les textures Kenney. Configure trois tuiles : `Grass`, `Mud`, `Steel`.
   - Attribuer les coordonnées (0,0), (1,0), (2,0).
2. **Détecter le terrain sous le tank**
   - Dans `Tank.gd`, ajoute :
     ```gdscript
     @export var terrain_map: TileMap

     func _get_tile_type() -> StringName:
         var cell := terrain_map.local_to_map(global_position)
         return terrain_map.get_cell_source_name(0, cell)
     ```
3. **Appliquer les effets**
   - Ajoute au `physics_process` :
     ```gdscript
     var tile := _get_tile_type()
     if tile == "Mud":
         velocity *= 0.6
         resource_manager.dissipate_heat(3.0 * delta)
         resource_manager.consume_fuel(0.05)
     elif tile == "Steel":
         resource_manager.add_heat(2.0 * delta)
     else:
         resource_manager.consume_fuel(0.1 * delta)
     ```
   - Ajoute un son `res://audio/ricochet.wav` (télécharge <https://freesound.org/people/InspectorJ/sounds/411791/>). Joue-le quand un projectile touche `Steel` (utilise `area_entered`).
4. **Surchauffe**
   - Déclare dans `Tank.gd` (en haut du script) : `var _overheated_until_msec: int = 0`.
   - Si `resource_manager.heat >= ResourceManager.HEAT_MAX`, empêche le tir pendant 3 s :
     ```gdscript
     func try_fire() -> void:
         var now := Time.get_ticks_msec()
         if now < _overheated_until_msec:
             return
         if resource_manager.heat >= ResourceManager.HEAT_MAX:
             _overheated_until_msec = now + 3000
             print("OVERHEAT")
             return
     ```

### Tests & validations
- Marche sur la boue → log `tile=Mud` (ajoute `print` temporaire et capture dans le rapport).
- Tire 30 fois d'affilée → message `OVERHEAT` + cooldown.
- Ajoute un test gdUnit4 `test_overheat.gd` pour vérifier `_overheated_until` (instancie Tank, simule heat).

### Critères d'acceptation
- Le tank ne glisse pas en dehors de la map (limite via `Rect2` clamp).
- Les sons sont importés en `Stream` (pas `Sample`).

## Sprint 4 — Refuel/Reload events + UI warnings

### Résultat attendu
Des événements de ravitaillement apparaissent, permettant de recharger Fuel/Ammo et d'afficher des alertes visuelles.

### Checklist
1. **Créer un système d'événements**
   - `scripts/systems/LogisticsEvent.gd` (Resource) avec propriétés : `event_type` (`refuel`, `reload`), `duration`, `reward`.
   - `scripts/systems/LogisticsDirector.gd` : gère un timer déclenchant un event toutes 120 s.
2. **Spawner un convoi**
   - `scenes/core/Convoy.tscn` (KinematicBody2D + Sprite). Il se dirige vers le tank.
   - Quand le tank touche le convoi (`body_entered`), appeler `resource_manager.refill_ammo(20)` et `resource_manager.regenerate_fuel(40)`.
   - Détruire le convoi (`queue_free`) et émettre `EventBus.emit_event("LOGISTICS_COMPLETED", {"type": current_event.event_type})`.
3. **UI warnings**
   - Ajoute dans HUD un `Label` rouge `WarningLabel` (texte vide par défaut).
   - Connecte les signaux `fuel_changed`/`ammo_changed` pour afficher `LOW FUEL` si < 25% et `LOW AMMO` si < 10.
   - Utilise `Tween` pour clignoter (Opacity 0↔1).
4. **Tests**
   - Crée un test `test_logistics_event.gd` vérifiant que `LogisticsDirector` crée un événement toutes 120 s (mock `delta`).
   - Vérifie manuellement qu'après 2 minutes un convoi spawn (utilise `Engine.time_scale = 10` pour accélérer durant le test).

### Critères d'acceptation
- Pas plus d'un convoi actif simultanément.
- Les alertes UI se coupent une fois la ressource restaurée.
- Ajoute `docs/previews/sprint4_refuel.gif` montrant la récupération de ressources.

## Dépendances
- Nécessite E1 complet.
- Fournit `ResourceManager` et `Tank` à E3, E4 et E5.

## Risques & mitigations
- **Balance injuste** : stocke les constantes dans `GameConfig` pour centraliser.
- **Entrées manette** : teste avec `godot --path ./game --run --display-driver headless` + `--input` (Godot CLI) si pas de manette.
- **Assets manquants** : si un lien Kenney est cassé, télécharge depuis la copie GitHub officielle `https://github.com/kenneyNL/TopdownTanksRedux`.
