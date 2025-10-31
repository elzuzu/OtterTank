---
id: E3
name: "EPIC E3 — Horde & IA ennemie"
kind: epic
status: todo
owners:
  - engineering
summary: "Créer les ennemis de base, leur IA de déplacement, la gestion des carcasses persistantes et les spawners adaptatifs de vague."
dependencies:
  - E1
  - E2
critical_path: true
---

# EPIC E3 — Horde & IA ennemie

## Objectif pédagogique
Tu dois fabriquer une IA d'ennemis capable de poursuivre le tank, de subir des dégâts et de laisser une carcasse bloquante. On ne veut pas de comportement improvisé : chaque comportement est décrit ici.

## Préparation commune
- Vérifie que `Tank.tscn`, `ResourceManager` et `EventBus` fonctionnent (tests E2 OK).
- Télécharge les sprites d'ennemis (Kenney Topdown Tanks Redux, `tankBody_darkLarge_outline.png` pour mini-boss, `tankBody_red_outline.png` pour grunt).
- Ajoute `docs/reference/ai_flowchart.png` (schéma que tu dessines avec draw.io reprenant les états). Le fichier doit montrer : `Spawn → Seek → Attack → Die → Carcass`.

## Sprint 1 — Ennemi de base (grunts)

### Résultat attendu
Un ennemi simple suit le tank, tire à cadence lente et meurt en laissant un corps qui bloque le pathfinding.

### Checklist chronologique
1. **Scène enemy**
   - `res://scenes/enemies/EnemyGrunt.tscn` : `CharacterBody2D` avec `Sprite2D`, `CollisionShape2D`, `Timer` (`FireTimer`).
   - Script `res://scripts/enemies/EnemyGrunt.gd` :
     ```gdscript
     extends CharacterBody2D

     @export var move_speed: float = 140.0
     @export var fire_interval: float = 1.2
     @export var projectile_scene: PackedScene = preload("res://scenes/core/ProjectileEnemy.tscn")

     var target: Node2D
     var health: float = 40.0

     func _ready() -> void:
         FireTimer.wait_time = fire_interval
         FireTimer.start()
         target = get_tree().get_first_node_in_group("player")
         add_to_group("enemies")

     func _physics_process(delta: float) -> void:
         if not target:
             return
         var direction := (target.global_position - global_position).normalized()
         velocity = direction * move_speed
         move_and_slide()

     func _on_FireTimer_timeout() -> void:
         if not target:
             return
         var projectile := projectile_scene.instantiate()
         projectile.global_position = global_position
         projectile.look_at(target.global_position)
         get_tree().current_scene.add_child(projectile)
         EventBus.emit_event("ENEMY_FIRED", {"enemy": self})

     func apply_damage(amount: float) -> void:
         health -= amount
         if health <= 0:
             _die()

     func _die() -> void:
         EventBus.emit_event("ENEMY_DIED", {"position": global_position})
         queue_free()
     ```
   - Ajoute un `Area2D` enfant `Hitbox` pour collision avec projectiles du joueur (`group` = `enemy_hitbox`).
2. **Projectile ennemi**
   - `res://scenes/core/ProjectileEnemy.tscn` (Area2D + collision + script direction). Dommages 10.
3. **Détection collisions**
   - Dans `Projectile.gd` (du joueur), ajoute :
     ```gdscript
     func _on_area_entered(area: Area2D) -> void:
         if area.is_in_group("enemy_hitbox"):
             area.get_parent().apply_damage(20)
             queue_free()
     ```
   - Connecte le signal `area_entered` via l'éditeur.
4. **Ajout dans Game**
   - Le `Tank` doit être dans le groupe `player` (dans `Main.gd` ou `Tank.tscn`).
   - Dans `Game.gd`, instancie un grunt toutes les 5 s (temporaire) :
     ```gdscript
     @onready var _spawn_timer := Timer.new()

     func _ready() -> void:
         add_child(_spawn_timer)
         _spawn_timer.wait_time = 5.0
         _spawn_timer.timeout.connect(_spawn_enemy)
         _spawn_timer.start()

     func _spawn_enemy() -> void:
         var enemy := preload("res://scenes/enemies/EnemyGrunt.tscn").instantiate()
         enemy.global_position = Vector2(randf_range(-400, 400), randf_range(-400, 400))
         add_child(enemy)
     ```

### Tests & validations
- Vérifie qu'un ennemi suit bien le tank en utilisant `Print` sur la distance.
- Tuer l'ennemi → `EventBus` log `ENEMY_DIED`.
- Ajoute test gdUnit4 `test_enemy_takes_damage.gd` : instancie l'ennemi, appelle `apply_damage(50)` → ennemi queue_free (utilise `yield` sur `tree.process_frame`).

### Critères d'acceptation
- L'ennemi est ajouté au groupe `enemies`.
- Pas de `Timer` en autostart manquant.

## Sprint 2 — Carcasses persistantes

### Résultat attendu
Quand un ennemi meurt, un node `Carcass` est instancié et reste sur le terrain en bloquant navigation et projectiles légers.

### Checklist
1. **Créer Carcass.tscn**
   - `StaticBody2D` avec `CollisionShape2D` large, `Sprite2D` (texture grise). Ajoute script :
     ```gdscript
     extends StaticBody2D

     @export var decay_time: float = 30.0

     func _ready() -> void:
         var timer := Timer.new()
         timer.wait_time = decay_time
         timer.one_shot = true
         add_child(timer)
         timer.timeout.connect(_on_timer_timeout)
         timer.start()

     func _on_timer_timeout() -> void:
         queue_free()
     ```
2. **Instanciation automatique**
   - Dans `EnemyGrunt._die()`, remplace par :
     ```gdscript
     var carcass := preload("res://scenes/enemies/Carcass.tscn").instantiate()
     carcass.global_position = global_position
     get_tree().current_scene.add_child(carcass)
     EventBus.emit_event("ENEMY_DIED", {"position": global_position})
     queue_free()
     ```
3. **Collision projectile**
   - Dans `Projectile.gd`, si collision avec `Carcass` (`body_entered`), `bounce` (changer direction) :
     ```gdscript
     func _on_body_entered(body: Node) -> void:
         if body.is_in_group("carcass"):
             direction = direction.bounce(body.global_position.direction_to(global_position))
             EventBus.emit_event("PROJECTILE_RICOCHET", {"position": global_position})
     ```
   - Ajoute le groupe `carcass` à `Carcass.tscn`.
4. **Persistant entre vagues**
   - Crée `CarcassManager.gd` (singleton autoload) stockant toutes les positions dans un `Array`. Lors d'un reload de scène, réinstancie les carcasses.
   - Ajoute test `test_carcass_persists.gd` (sauvegarde en Resource `res://tests/tmp_save.tres`).

### Tests & validations
- Tue 5 ennemis : 5 carcasses doivent rester en place pendant 30 s (chronomètre).
- Les projectiles du joueur ricochets (vérifie par visuel et log `PROJECTILE_RICOCHET`).

### Critères d'acceptation
- Les carcasses ne génèrent pas plus de 5 draw calls supplémentaires (profil via `Rendering > Diagnostics`).
- Les carcasses sont ajoutées à un pool `ObjectPool` (intègre avec E1 pour éviter allocations).

## Sprint 3 — Spawner & vagues adaptatives

### Résultat attendu
Un spawner lit les données du `ResourceManager` et du `Telemetry` pour ajuster le nombre d'ennemis. La carte est découpée en chunks de 64x64.

### Checklist
1. **Influence Grid**
   - `scripts/systems/InfluenceGrid.gd` : une grille 2D (taille 128x128). Fournit `set_value(x, y, value)` et `get_value(x, y)`.
   - Chaque fois qu'un ennemi meurt, augmente la valeur autour de la cellule (dissuade spawn).
2. **Spawner**
   - `scripts/systems/EnemySpawner.gd` :
     ```gdscript
     extends Node

     @export var spawn_interval: float = 3.0
     @export var max_enemies: int = 50

     @onready var _timer := Timer.new()

     func _ready() -> void:
         add_child(_timer)
         _timer.wait_time = spawn_interval
         _timer.timeout.connect(_spawn)
         _timer.start()

     func _spawn() -> void:
         if get_tree().get_nodes_in_group("enemies").size() >= max_enemies:
             return
         var spawn_pos := _pick_spawn_position()
         var enemy := preload("res://scenes/enemies/EnemyGrunt.tscn").instantiate()
         enemy.global_position = spawn_pos
         get_tree().current_scene.add_child(enemy)
     ```
   - `_pick_spawn_position()` doit éviter la zone 256x256 autour du tank et les cellules d'influence > 5.
3. **Adaptation**
   - Ajuste `spawn_interval` dynamiquement :
     ```gdscript
     func adjust_to_player_state(manager: ResourceManager) -> void:
         if manager.fuel < 20:
             _timer.wait_time = 6.0
         elif manager.heat > 80:
             _timer.wait_time = 4.0
         else:
             _timer.wait_time = 3.0
     ```
   - Appelle cette fonction toutes les 5 s depuis `Game.gd` (Timer séparé).
4. **Mini-boss**
   - Au bout de 3 minutes, spawn `EnemyMiniBoss.tscn` (avec double HP, tir multiple). Documente ses stats dans `docs/data/enemies.csv`.

### Tests & validations
- Ajoute logs `print("Spawn interval", _timer.wait_time)` pour prouver l'adaptation.
- Test `test_spawner_respects_cap.gd` : simule 51 ennemis → `_spawn()` ne dépasse pas `max_enemies`.
- Profil Godot → 300 ennemis + 500 projectiles -> >45 FPS (documente dans `docs/perf/sprint3.md`).

### Critères d'acceptation
- InfluenceGrid stockée dans `Telemetry` (`Telemetry.set_gauge("influence_avg", moyenne)`).
- Les mini-boss laissent une carcasse plus grosse (utilise `CarcassLarge.tscn`).

## Dépendances
- Utilise `ResourceManager` (E2) et `ObjectPool` (E1).
- Alimente E4 (procédural) et E7 (director) via InfluenceGrid.

## Risques & mitigations
- **Surcharge CPU** : active `physics/common/physics_fps=120` pour test stress.
- **Collisions instables** : applique `set_deferred("disabled", true)` lors de la mort pour éviter un double hit.
- **Mémoire** : libère carcasses après 60 s si > 30 actives (`CarcassManager`).
