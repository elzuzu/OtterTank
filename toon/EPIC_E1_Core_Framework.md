---
id: E1
name: "EPIC E1 — Core & Framework"
kind: epic
status: todo
owners:
  - engineering
summary: "Mettre en place la boucle de jeu Godot, l'infrastructure d'événements, le pooling et les outils debug pour supporter l'ensemble des autres epics."
dependencies: []
critical_path: true
---

# EPIC E1 — Core & Framework

## Objectif pédagogique (tu ne dois pas réfléchir)
Tu dois **copier-coller** les étapes ci-dessous dans l'ordre. Ne saute jamais une case. Quand on te demande de créer un fichier, tu dois utiliser exactement le chemin et le contenu fournis. Si tu as un doute, recommence la commande.

## Prérequis exacts
- Installer **Godot 4.1.3 LTS** depuis <https://godotengine.org/download>. Vérifie la somme SHA256 fournie sur la page officielle (copie la valeur affichée, lance `shasum -a 256 Godot_v4.1.3-stable_linux.x86_64.zip` et compare).
- Installer **Python 3.10** (nécessaire pour gdUnit4 CLI) et **Node.js 18** (pour TOON plus tard). Vérifie avec `python3 --version` et `node --version`.
- Vérifier que `git` est configuré (`git config user.name`, `git config user.email`).

## Sprint 1 — Initialisation du projet Godot

### Résultat attendu
Un projet Godot minimal dans `/game` avec scripts de lancement multiplateforme et README mis à jour. Tu dois pouvoir lancer `godot --headless --path ./game --run` sans erreur.

### Checklist chronologique
1. **Créer la racine Godot**
   1. Supprime le dossier `/game` s'il existe : `rm -rf game`.
   2. Lance `godot --headless --path . --editor` puis fais `Project > New`.
   3. Chemin exact : `/workspace/OtterTank/game`. Nom du projet : `OtterTank`.
2. **Configurer les Project Settings** *(tu dois cocher exactement ce qu'on dit)*
   - `Project > Project Settings > Rendering > Textures > Default Texture Filter = Nearest`
   - `Project > Project Settings > Rendering > Textures > Default Texture Repeat = Enabled`
   - `Project > Project Settings > Display > Window > Size > Width = 1920`, `Height = 1080`
   - `Project > Project Settings > Display > Window > Stretch > Mode = canvas_items`, `Aspect = keep`
   - `Project > Project Settings > General > 2d > Use Pixel Snap = On`
   - Sauvegarde (`Ctrl+S`).
3. **Activer les autoloads placeholders**
   - `Project > Project Settings > AutoLoad` → ajoute `res://autoload/EventBus.gd` alias `EventBus` (créé vide pour l'instant).
   - Ajoute `res://autoload/Telemetry.gd` alias `Telemetry`.
   - Ajoute `res://autoload/GameConfig.gd` alias `GameConfig`.
   - Si les scripts n'existent pas encore, crée des fichiers vides via `FileSystem` (clic droit > New Script). Le contenu provisoire est donné plus bas.
4. **Organiser l'arborescence** *(copie-colle cette commande)*
   ```bash
   mkdir -p game/{autoload,scenes/{core,ui},scripts/{core,systems},tests,tools} build/{debug,release}
   ```
5. **Ajouter les scripts utilitaires**
   - Crée `tools/run_game.sh` avec ce contenu exact :
     ```bash
     #!/usr/bin/env bash
     set -euo pipefail
     cd "$(dirname "$0")/.."
     godot --path ./game --run
     ```
   - Crée `tools/run_editor.sh` avec :
     ```bash
     #!/usr/bin/env bash
     set -euo pipefail
     cd "$(dirname "$0")/.."
     godot --path ./game --editor
     ```
   - Rendez-les exécutables : `chmod +x tools/run_game.sh tools/run_editor.sh`.
6. **Mettre à jour `.gitignore`**
   - Ajoute si absent :
     ```
     .godot/
     build/
     export_presets.cfg
     ```
7. **README**
   - Ajoute une section "Lancer Godot" avec les commandes `./tools/run_editor.sh` et `./tools/run_game.sh`.

### Tests & validations obligatoires
- `godot --headless --path ./game --run` doit afficher `Running: res://main.tscn` puis `No main scene has ever been defined` (c'est OK à cette étape).
- `./tools/run_editor.sh` ouvre l'éditeur sans crash.
- `tree -L 2 game` doit afficher l'arborescence créée (copie la sortie dans le ticket).

### Critères d'acceptation concrets
- `project.godot` présent et contient `[application] config/name="OtterTank"`.
- Les scripts d'autoload vides existent (voir section gabarits). Ne laisse pas de fichier `.tmp`.

### Gabarits (copie-colle tel quel)
`game/autoload/EventBus.gd`
```gdscript
extends Node

func _ready() -> void:
    pass # rempli dans Sprint 3
```

`game/autoload/Telemetry.gd`
```gdscript
extends Node

var _counters: Dictionary = {}

func _ready() -> void:
    _counters.clear()
```

`game/autoload/GameConfig.gd`
```gdscript
extends Resource

const TARGET_FPS: int = 60
const MAX_PROJECTILES: int = 500
const MAX_ENEMIES: int = 300
```

## Sprint 2 — Boucle de jeu et scène principale

### Résultat attendu
Scènes `Main.tscn`, `Game.tscn`, `CameraRig.tscn` et scripts associés générant une boucle fixe à 60 Hz, avec InputMap configurée pour clavier et manette.

### Checklist chronologique
1. **Créer `Main.tscn`**
   - Dans Godot, `FileSystem > scenes/core > New Scene > Node`. Renomme en `Main`. Sauvegarde `res://scenes/core/Main.tscn`.
   - Attache un script via `Attach Script` → chemin `res://scripts/core/Main.gd` → contenu :
     ```gdscript
     extends Node

     @onready var _game_scene: PackedScene = preload("res://scenes/core/Game.tscn")

     func _ready() -> void:
         var game_instance := _game_scene.instantiate()
         add_child(game_instance)
     ```
2. **Créer `Game.tscn`**
   - Scene root `Node` nommé `Game` avec script `res://scripts/core/Game.gd` :
     ```gdscript
     extends Node

     signal tick_started(delta: float)
     signal tick_ended(delta: float)

     const FIXED_DELTA := 1.0 / GameConfig.TARGET_FPS

     var _accumulator := 0.0

     func _ready() -> void:
         Engine.physics_ticks_per_second = GameConfig.TARGET_FPS
         set_process(true)
         set_physics_process(true)

     func _process(delta: float) -> void:
         _accumulator += delta
         while _accumulator >= FIXED_DELTA:
             _accumulator -= FIXED_DELTA
             emit_signal("tick_started", FIXED_DELTA)
             _on_fixed_update(FIXED_DELTA)
             emit_signal("tick_ended", FIXED_DELTA)

     func _physics_process(delta: float) -> void:
         pass # placeholder

     func _on_fixed_update(fixed_delta: float) -> void:
         pass # sera rempli plus tard
     ```
   - Ajoute un `Node2D` enfant nommé `CameraRigRoot` avec une instance de `CameraRig.tscn` (créée à l'étape suivante).
3. **Créer `CameraRig.tscn`**
   - `Node2D` racine `CameraRig` avec script `res://scripts/core/CameraRig.gd` :
     ```gdscript
     extends Node2D

     @export var zoom_speed: float = 3.0
     @onready var _camera: Camera2D = $Camera2D

     func set_zoom_level(level: float) -> void:
         _camera.zoom = Vector2(level, level)

     func apply_shake(intensity: float, duration: float) -> void:
         # placeholder: tu ajoutes un TODO pour Sprint 5 si besoin
         pass
     ```
   - Ajoute un `Camera2D` enfant, `Current = On`, `Zoom = (1,1)`.
4. **Configurer les autoloads**
   - Vérifie dans `Project Settings > AutoLoad` que `EventBus.gd`, `Telemetry.gd`, `GameConfig.gd` sont cochés.
5. **Configurer l'InputMap** *(copie ces actions)*
   - `Project > Project Settings > Input Map` → `+` → ajoute actions `move_up`, `move_down`, `move_left`, `move_right`, `shoot`, `dash`, `aim_up`, `aim_down`, `aim_left`, `aim_right`.
   - Associe : `move_up` = `W` + `DPad Up`; `move_down` = `S` + `DPad Down`; `move_left` = `A` + `DPad Left`; `move_right` = `D` + `DPad Right`.
   - `shoot` = `Space` + `Joypad Button 7 (Xbox RB)`; `dash` = `Left Shift` + `Joypad Button 0 (A)`.
   - Pour `aim_*`, utilise `Joypad Motion` axes (X/Y).
6. **Définir la scène principale**
   - `Project > Project Settings > Application > Run` → `Main Scene = res://scenes/core/Main.tscn`.

### Tests & validations obligatoires
- `godot --headless --path ./game --run` doit afficher `GameLoop ready` (ajoute `print("GameLoop ready")` dans `_ready()` pour confirmer puis laisse-le).
- Dans l'éditeur, `Debug > Visible Collision Shapes` actif : aucun warning.
- `Project > Project Settings > Input Map > Export` (bouton `Copy to Clipboard`). Colle la configuration dans le journal de sprint pour preuve.

### Critères d'acceptation
- `Game.gd` expose les signaux `tick_started` et `tick_ended` (vérifié dans le script).
- La scène `CameraRig.tscn` est sauvegardée dans `scenes/core`.
- Aucun `TODO` non justifié.

## Sprint 3 — EventBus, ObjectPool et Telemetry

### Résultat attendu
Implémentations fonctionnelles d'un bus d'événements, d'un pool générique et d'une télémétrie affichant un overlay debug.

### Checklist chronologique
1. **EventBus** (`game/autoload/EventBus.gd`)
   - Remplace le contenu par :
     ```gdscript
     extends Node

     signal event_emitted(event_name: StringName, payload: Dictionary)

     var _listeners: Dictionary = {}

     func subscribe(event_name: StringName, target: Object, method: StringName) -> void:
         if not _listeners.has(event_name):
             _listeners[event_name] = []
         var entry := {"target": target, "method": method}
         _listeners[event_name].append(entry)

     func unsubscribe(event_name: StringName, target: Object, method: StringName) -> void:
         if not _listeners.has(event_name):
             return
         _listeners[event_name] = _listeners[event_name].filter(func(item):
             return not (item.target == target and item.method == method)
         )

     func emit_event(event_name: StringName, payload: Dictionary = {}) -> void:
         if _listeners.has(event_name):
             for item in _listeners[event_name]:
                 if is_instance_valid(item.target):
                     item.target.call_deferred(item.method, payload)
         emit_signal("event_emitted", event_name, payload)
     ```
2. **ObjectPool** (`game/scripts/systems/ObjectPool.gd`)
   ```gdscript
   extends Node

   class_name ObjectPool

   var _scenes: Dictionary = {}
   var _pool: Dictionary = {}

   func preload_scene(key: StringName, scene_path: String) -> void:
         _scenes[key] = load(scene_path)
         _pool[key] = []

   func acquire(key: StringName) -> Node:
         if not _scenes.has(key):
             push_error("Scene key %s not registered" % key)
             return null
         if _pool[key].is_empty():
             return _scenes[key].instantiate()
         return _pool[key].pop_back()

   func release(key: StringName, node: Node) -> void:
         if not _pool.has(key):
             push_error("Pool key %s not registered" % key)
             return
         node.queue_free() if not node.is_inside_tree() else node.remove_from_parent()
         node.set_process(false)
         node.visible = false if node.has_method("set_visible") else node.visible
         _pool[key].append(node)
   ```
3. **Telemetry** (`game/autoload/Telemetry.gd`)
   ```gdscript
   extends Node

   var _counters: Dictionary = {}
   var _gauges: Dictionary = {}

   func register_counter(name: StringName) -> void:
         _counters[name] = 0

   func increment(name: StringName, value: int = 1) -> void:
         if not _counters.has(name):
             register_counter(name)
         _counters[name] += value

   func register_gauge(name: StringName) -> void:
         _gauges[name] = 0.0

   func set_gauge(name: StringName, value: float) -> void:
         if not _gauges.has(name):
             register_gauge(name)
         _gauges[name] = value

   func dump() -> Dictionary:
         return {"counters": _counters.duplicate(), "gauges": _gauges.duplicate()}
   ```
4. **DebugOverlay**
   - Crée `scenes/ui/DebugOverlay.tscn` (CanvasLayer) avec `MarginContainer > VBoxContainer > Label` (3 labels : `FpsLabel`, `ProjectilesLabel`, `EnemiesLabel`).
   - Script `game/scripts/ui/DebugOverlay.gd` :
     ```gdscript
     extends CanvasLayer

     func _ready() -> void:
         set_process(true)

     func _process(_delta: float) -> void:
         $MarginContainer/VBoxContainer/FpsLabel.text = "FPS: %d" % Engine.get_frames_per_second()
         $MarginContainer/VBoxContainer/ProjectilesLabel.text = "Projectiles: %d" % Telemetry._counters.get("projectiles", 0)
         $MarginContainer/VBoxContainer/EnemiesLabel.text = "Enemies: %d" % Telemetry._counters.get("enemies", 0)
     ```
   - Dans `Game.tscn`, ajoute `DebugOverlay` comme enfant.
5. **Tests automatiques**
   - Installe [gdUnit4](https://mikeschulze.github.io/gdUnit4/) : `godot --headless --path ./game --quit-after 1 --script res://addons/gdUnit4/scripts/install_gdUnit4.gd`.
   - Crée `game/tests/test_object_pool.gd` :
     ```gdscript
     extends Node
     class_name TestObjectPool

     var pool := ObjectPool.new()

     func before() -> void:
         pool.preload_scene("dummy", "res://tests/dummy/TestNode.tscn")

     func test_acquire_and_release() -> void:
         var first := pool.acquire("dummy")
         assert_not_null(first)
         pool.release("dummy", first)
         var second := pool.acquire("dummy")
         assert_true(first == second)
     ```
   - Crée la scène `game/tests/dummy/TestNode.tscn` (Node vide) pour le test.

### Tests & validations obligatoires
- `godot --headless --path ./game --run` → l'overlay affiche FPS dans la console (utilise `print`).
- `godot --headless --path ./game --script res://addons/gdUnit4/bin/gdUnit4.gd -s` exécute les tests et doit se terminer par `0 failed`.
- Exporte un dump télémétrie : dans la console Godot, exécute `print(Telemetry.dump())` → doit renvoyer un dictionnaire.

### Critères d'acceptation
- Aucun warning `push_error` dans la console lorsque tu acquiers/libères 10 fois.
- `DebugOverlay` se toggle avec `F3` (ajoute `Input.is_action_just_pressed("toggle_debug")`).
- Les tests gdUnit4 sont ajoutés au pipeline CI (fichier `.github/workflows/ci.yml` sera créé en Sprint 4).

## Sprint 4 — Tooling, CI et packaging

### Résultat attendu
Overlay toggle, scripts de build multiplateforme, pipeline GitHub Actions qui lance les tests et exporte un build debug Linux.

### Checklist
1. **Action GitHub**
   - Crée `.github/workflows/godot-ci.yml` avec le pipeline suivant :
     ```yaml
     name: Godot CI

     on:
       push:
         branches: [ main ]
       pull_request:
         branches: [ main ]

     jobs:
       build:
         runs-on: ubuntu-latest
         steps:
           - uses: actions/checkout@v3
           - uses: chickensoft-games/setup-godot@v1
             with:
               version: "4.1.3"
               include-templates: true
           - name: Run tests
             run: godot --headless --path ./game --script res://addons/gdUnit4/bin/gdUnit4.gd -s
           - name: Export Linux debug
             run: godot --headless --path ./game --export-debug "Linux/X11" build/debug/OtterTank.x86_64
           - uses: actions/upload-artifact@v3
             with:
               name: ottertank-linux-debug
               path: build/debug/OtterTank.x86_64
     ```
2. **Toggle DebugOverlay**
   - Dans `Game.gd`, ajoute :
     ```gdscript
     @onready var _debug_overlay := $DebugOverlay

     func _ready() -> void:
         Engine.physics_ticks_per_second = GameConfig.TARGET_FPS
         InputMap.add_action("toggle_debug")
         InputMap.action_add_event("toggle_debug", InputEventKey.new().tap(func(event):
             event.keycode = Key.F3
         ))
         set_process(true)
         set_physics_process(true)

     func _input(event: InputEvent) -> void:
         if event.is_action_pressed("toggle_debug"):
             _debug_overlay.visible = not _debug_overlay.visible
     ```
3. **Scripts de build**
   - Ajoute `tools/build_debug.sh` :
     ```bash
     #!/usr/bin/env bash
     set -euo pipefail
     cd "$(dirname "$0")/.."
     godot --headless --path ./game --export-debug "Linux/X11" build/debug/OtterTank.x86_64
     ```
   - Ajoute `tools/build_release.sh` similaire (`--export-release` vers `build/release/OtterTank.x86_64`).
4. **Documentation**
   - Dans `README.md`, ajoute un tableau "Commandes" : `run_editor`, `run_game`, `build_debug`, `build_release`, `test_gdunit`.

### Tests & validations
- `chmod +x tools/*.sh`.
- `./tools/build_debug.sh` génère un binaire dans `build/debug/` (vérifie avec `file build/debug/OtterTank.x86_64`).
- Pipeline GitHub doit réussir localement : exécute `act -j build` (si `act` n'est pas dispo, indique-le et teste manuellement chaque step).

### Critères d'acceptation
- Build debug < 150 Mo.
- README contient captures d'écran CLI (copie des commandes et sorties principales).
- Aucun script ne contient de `TODO` ou `FIXME` non résolu.

## Risques & mitigations
- **Godot non installé** : vérifie `which godot` au début de chaque sprint.
- **Autoload manquant** : si `EventBus` n'est pas trouvé, retourne au Sprint 1 et recommence la configuration.
- **CI lente** : active le cache `~/.cache/godot` si nécessaire (ajouter `actions/cache`).

## Dépendances inter-epics
- Tous les epics suivants (E2 à E9) nécessitent cette base. Si un test échoue ici, n'avance pas.
