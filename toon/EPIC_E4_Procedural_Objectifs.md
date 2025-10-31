---
id: E4
name: "EPIC E4 — Procédural & Objectifs"
kind: epic
status: todo
owners:
  - engineering
summary: "Assembler les niveaux procéduraux chunk-based, intégrer les objectifs (escorter, défendre, détruire, hacker) et synchroniser les événements avec le director."
dependencies:
  - E1
  - E2
  - E3
critical_path: true
---

# EPIC E4 — Procédural & Objectifs

## Objectif pédagogique
Tu construis un générateur de niveau simple mais déterministe basé sur des chunks 64x64, puis tu implémentes quatre objectifs jouables. Chaque action est décrite mot pour mot, ne saute rien.

## Préparation commune
- Installe l'outil `toon` (pour visualiser la roadmap) : `npm install -g @johannschopplich/toon` (vérifie `toon --version`).
- Télécharge le pack `Kenney Modular 2D Blocks` pour obtenir divers chunks. Place-les dans `game/assets/chunks/`.
- Crée un dossier `docs/procgen/` qui contiendra :
  - `biomes.md` (table des chunks disponibles).
  - `objectives.md` (description textuelle + conditions de victoire).

## Sprint 1 — Librairie de chunks

### Résultat attendu
Un système peut charger des chunks `.tscn` depuis `res://scenes/chunks/` et générer une carte 5x5 alignée.

### Checklist
1. **Créer la structure**
   - Commande :
     ```bash
     mkdir -p game/scenes/chunks/{common,desert,urban,forest}
     mkdir -p game/scripts/procgen
     ```
2. **Chunk format**
   - Chaque chunk est une scène `Node2D` contenant un `TileMap` + des `Marker2D` (`SpawnPoint`, `ObjectiveSocket`).
   - Crée `Chunk_Forest_Crossroad.tscn` comme exemple (TileMap 64x64, collisions configurées).
   - Ajoute un script `ChunkDefinition.gd` :
     ```gdscript
     extends Node2D

     @export var tags: Array[StringName]
     @export var difficulty: int = 1

     func get_spawn_markers() -> Array[Marker2D]:
         return get_tree().get_nodes_in_group("chunk_spawn")
     ```
   - Tous les markers pour spawn doivent être ajoutés au groupe `chunk_spawn`.
3. **Catalogue**
   - Crée `game/scripts/procgen/ChunkCatalogue.gd` :
     ```gdscript
     extends Node

     class_name ChunkCatalogue

     var _chunks_by_biome: Dictionary = {}

     func register_chunk(biome: StringName, scene_path: String) -> void:
         if not _chunks_by_biome.has(biome):
             _chunks_by_biome[biome] = []
         _chunks_by_biome[biome].append(scene_path)

     func pick_chunk(biome: StringName, required_tags: Array[StringName] = []) -> PackedScene:
         var candidates := _chunks_by_biome.get(biome, [])
         var filtered := []
         for path in candidates:
             var scene := load(path)
             var instance := scene.instantiate()
             var definition := instance as Node2D
             var has_all := true
             for tag in required_tags:
                 if tag not in definition.tags:
                     has_all = false
                     break
             if has_all:
                 filtered.append(scene)
         if filtered.is_empty():
             push_error("No chunk with tags %s" % required_tags)
             return load(candidates[0])
         return filtered[randi() % filtered.size()]
     ```
4. **Enregistrement automatique**
   - Ajoute un script `autoload/ChunkRegistry.gd` qui scanne `res://scenes/chunks` à `_ready()` et appelle `ChunkCatalogue.register_chunk()`.
   - Documente la liste dans `docs/procgen/biomes.md` (table Markdown Biome/Chunk/Tags/Difficulté).

### Tests & validations
- Exécute un test gdUnit `test_chunk_catalogue.gd` qui enregistre 3 chunks factices et vérifie la sélection par tag.
- Capture d'écran Godot montrant un chunk instancié dans la scène (sauvegarde `docs/procgen/chunk_preview.png`).

### Critères d'acceptation
- Chaque chunk contient au moins 2 `Marker2D` (spawn + objectif).
- Aucun chunk ne dépasse 128 nodes (profil via `Scene > Information`).

## Sprint 2 — Générateur de niveau

### Résultat attendu
Une classe `LevelGenerator` assemble une grille 5x5, connecte les chunks et marque un chemin critique `Start → Mid → Boss`.

### Checklist
1. **Scripts**
   - `game/scripts/procgen/LevelBlueprint.gd` (Resource) :
     ```gdscript
     extends Resource

     @export var biome: StringName = "forest"
     @export var seed: int = 0
     @export var grid_size: Vector2i = Vector2i(5, 5)
     @export var objective_sequence: Array[StringName] = ["escort", "defend", "destroy", "hack"]
     ```
   - `game/scripts/procgen/LevelGenerator.gd` :
     ```gdscript
     extends Node

     class_name LevelGenerator

     @export var catalogue: ChunkCatalogue

     func generate(blueprint: LevelBlueprint) -> Array[Node2D]:
         randomize()
         var chunks := []
         for y in blueprint.grid_size.y:
             for x in blueprint.grid_size.x:
                 var required := []
                 if x == 0 and y == 0:
                     required = ["start"]
                 elif x == blueprint.grid_size.x - 1 and y == blueprint.grid_size.y - 1:
                     required = ["boss"]
                 var scene := catalogue.pick_chunk(blueprint.biome, required)
                 var instance := scene.instantiate()
                 instance.position = Vector2(x * 640, y * 640)
                 chunks.append(instance)
         return chunks
     ```
   - Ajoute un `LevelAssembler.gd` qui prend la liste et les ajoute à `Game`.
2. **Chemin critique**
   - Après génération, marque les chunks du chemin avec un `Sprite2D` (couleur bleu) pour debug.
   - Stocke le chemin dans `Telemetry` (`Telemetry.set_gauge("path_length", path.size())`).
3. **Seed**
   - Dans `LevelGenerator.generate`, utilise `RandomNumberGenerator` initialisé avec `blueprint.seed` pour reproductibilité.
4. **Tests**
   - `test_level_generator_seed.gd` : génère deux fois avec même seed → positions identiques.
   - `test_level_generator_tags.gd` : vérifie que la case (4,4) contient un chunk tagué `boss`.

### Critères d'acceptation
- Génération < 200 ms (mesure `Performance.get_monitor` ou `OS.get_ticks_msec`).
- Fichier `docs/procgen/objectives.md` contient l'ordre exact `Start`, `Objective1`, `Objective2`, `Boss`.

## Sprint 3 — Objectifs jouables

### Résultat attendu
Quatre objectifs distincts, chacun instancié via un `ObjectiveController` qui gère états, succès, échec et récompenses.

### Checklist
1. **Base Objective**
   - `scripts/objectives/ObjectiveBase.gd` :
     ```gdscript
     extends Node

     signal completed(success: bool)

     @export var time_limit: float = 0.0

     func start(objective_data: Dictionary) -> void:
         pass
     ```
2. **Escort**
   - `ObjectiveEscort.gd` : instancie un convoi (reuse `Convoy.tscn` E2). Conditions : convoi arrive sur `Marker2D` `EscortDestination`.
3. **Defend**
   - `ObjectiveDefend.gd` : place une `Structure` à protéger (HP). Utilise `EnemySpawner` pour vagues ciblées.
4. **Destroy**
   - `ObjectiveDestroy.gd` : spawn une usine (`Factory.tscn`) avec 3 points faibles (`Area2D`). Le joueur doit tirer 10 fois.
5. **Hack**
   - `ObjectiveHack.gd` : mini-jeu QTE (appuyer sur touches aléatoires). Utilise `InputEventKey`.
6. **ObjectiveManager**
   - `scripts/objectives/ObjectiveManager.gd` : lit `LevelBlueprint.objective_sequence`, instancie les controllers correspondants et écoute `completed`. Récompenses : `resource_manager.refill_ammo(10)` si succès, `Heat` +20 si échec.
7. **UI**
   - Ajoute un panneau `ObjectivePanel` dans le HUD affichant `Current Objective`, `Timer`, `Status`.

### Tests & validations
- Pour chaque objectif, capture un GIF (10 s) stocké dans `docs/previews/objective_*.gif`.
- Tests gdUnit : `test_objective_manager_sequence.gd` (mock controllers, s'assurer de l'ordre) et `test_objective_timeout.gd`.
- Vérifie manuellement que l'échec affiche `Objective Failed` en rouge.

### Critères d'acceptation
- Aucun objectif ne laisse de nodes orphelins (utilise `is_queued_for_deletion`).
- Les récompenses sont loggées via `EventBus.emit_event("OBJECTIVE_COMPLETED", {"type": objective})`.

## Sprint 4 — Synchronisation Director & Events

### Résultat attendu
Le générateur signale chaque étape au Director (E7) via `EventBus`, et les objectifs influencent l'`InfluenceGrid` (E3).

### Checklist
1. **EventBus**
   - Émissions requises :
     - `LEVEL_GENERATED` (payload `chunks_count`, `biome`).
     - `OBJECTIVE_STARTED` (`type`, `time_limit`).
     - `OBJECTIVE_COMPLETED` (`type`, `success`).
2. **InfluenceGrid**
   - Quand un objectif est réussi, appelle `InfluenceGrid.set_value` sur les cellules correspondantes (`+2`). Échec → `-3`.
3. **War Room Debug**
   - Crée une scène `scenes/ui/WarRoomPanel.tscn` affichant les événements reçus (ListView). Ajoute un bouton `Export Logs` qui écrit `user://logs/war_room.log`.
4. **Tests**
   - `test_eventbus_signals.gd` : s'abonne à `EventBus` et vérifie l'ordre des signaux.
   - `test_influence_updates.gd` : simule un objectif réussi et vérifie la grille.

### Critères d'acceptation
- Export logs <= 1 Mo par session.
- UI WarRoom accessible via touche `Tab` (ajoute action `toggle_war_room`).

## Dépendances
- Utilise Tank & ResourceManager (E2), EnemySpawner (E3).
- Fournit données au Director (E7) et à la meta (E6).

## Risques & mitigations
- **Proc-gén incohérente** : ajoute un test snapshot (sauvegarde `res://tests/snapshots/level_seed_42.tres`).
- **Objectifs buggés** : active `visible collision shapes` pendant QA.
- **Perf** : n'ajoute jamais plus de 25 nodes par chunk (utilise `tilemap.layer = 0` et collisions simplifiées).
