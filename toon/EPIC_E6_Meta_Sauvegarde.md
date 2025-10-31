---
id: E6
name: "EPIC E6 — Méta Progression & Sauvegarde"
kind: epic
status: todo
owners:
  - engineering
summary: "Implémenter la méta progression persistante (tech tree, atlas global), les sauvegardes chiffrées et les analytics de run."
dependencies:
  - E1
  - E2
  - E3
  - E4
  - E5
critical_path: true
---

# EPIC E6 — Méta Progression & Sauvegarde

## Objectif pédagogique
Tu dois créer la méta qui motive les reruns : sauvegarde chiffrée, tech tree, atlas de campagne persistant et analytics de run. Chaque section contient des scripts à copier-coller.

## Préparation commune
- Vérifie `T1 — KBM Gold Standard` : toutes les options sauvegardées doivent inclure les réglages KBM (sensibilité, inversion, indicateurs) sans introduire de presets manette implicites.
- Ajouter le plugin officiel Godot `godot-encrypt` (copie le dossier dans `game/addons/godot-encrypt/`).
- Créer `docs/meta/tech_tree.xlsx` (modèle tableur) avec colonnes `NodeID`, `Prerequisites`, `Cost`, `Effect`.
- Ajouter un dossier `saves/` à la racine git (gitignored) pour les tests.

## Sprint 1 — Système de sauvegarde chiffrée

### Résultat attendu
Sauvegarder/charger l'état méta (`TechTree`, `Atlas`, `Settings`) dans un fichier chiffré AES.

### Checklist
1. **Data model**
   - `scripts/meta/MetaState.gd` (Resource) :
     ```gdscript
     extends Resource

     class_name MetaState

     @export var tech_nodes: Dictionary = {}
     @export var atlas_state: Dictionary = {}
     @export var options: Dictionary = {
         "audio": {"master_db": 0},
         "video": {"resolution": Vector2i(1920, 1080)}
     }
     ```
2. **Serializer**
   - `scripts/meta/MetaSerializer.gd` :
     ```gdscript
     extends Node

     class_name MetaSerializer

     const SAVE_PATH := "user://saves/meta.save"
     const SECRET_KEY := "REMPLACE_CECI_PAR_UNE_CLE_32_BYTES"

     func save_state(state: MetaState) -> void:
         var dir := DirAccess.open("user://saves")
         if dir == null:
             DirAccess.make_dir_recursive("user://saves")
         var data := JSON.stringify({
             "tech_nodes": state.tech_nodes,
             "atlas_state": state.atlas_state,
             "options": state.options
         })
         var encrypted := Encryptor.encrypt_aes256(data.to_utf8_buffer(), SECRET_KEY)
         var file := FileAccess.open(SAVE_PATH, FileAccess.WRITE)
         file.store_buffer(encrypted)
         file.close()

     func load_state() -> MetaState:
         if not FileAccess.file_exists(SAVE_PATH):
             return MetaState.new()
         var file := FileAccess.open(SAVE_PATH, FileAccess.READ)
         var encrypted := file.get_buffer(file.get_length())
         var decrypted := Encryptor.decrypt_aes256(encrypted, SECRET_KEY)
         var parsed := JSON.parse_string(decrypted.get_string_from_utf8())
         var state := MetaState.new()
         state.tech_nodes = parsed.get("tech_nodes", {})
         state.atlas_state = parsed.get("atlas_state", {})
         state.options = parsed.get("options", state.options)
         return state
     ```
   - Stocke la clé dans `.env` (Sprint 4) et charge via `ProjectSettings.globalize_path` pour éviter commit de clé.
3. **Tests**
   - `test_meta_serializer_roundtrip.gd` : sauvegarde un état, recharge, compare `tech_nodes`.
4. **CI**
   - Ajoute une étape dans `godot-ci.yml` pour exécuter les tests meta.

### Critères d'acceptation
- Le fichier `user://saves/meta.save` existe après un run.
- Le contenu n'est pas lisible en clair (ouvre avec `cat`, tu dois voir du binaire).

## Sprint 2 — Tech Tree & progression

### Résultat attendu
Tech tree interactif avec nodes achetables, dépendances visuelles et effets appliqués au jeu (slots supplémentaires, regen, etc.).

### Checklist
1. **Data**
   - Remplis `docs/meta/tech_tree.xlsx` avec au moins 15 nodes. Export CSV `game/data/tech_tree.csv`.
2. **TechNode**
   - `scripts/meta/TechNode.gd` (Resource) : `id`, `cost`, `prerequisites` (`Array[String]`), `effects` (`Dictionary`).
3. **TechTreeManager**
   - `scripts/meta/TechTreeManager.gd` :
     ```gdscript
     extends Node

     class_name TechTreeManager

     var nodes: Dictionary = {}
     var owned_nodes: Dictionary = {}

     func load_from_csv(path: String) -> void:
         var file := FileAccess.open(path, FileAccess.READ)
         file.get_line() # skip header
         while not file.eof_reached():
             var columns := file.get_csv_line()
             if columns.size() < 4:
                 continue
             nodes[columns[0]] = {
                 "cost": columns[1].to_int(),
                 "prereq": columns[2].split(";", false),
                 "effects": JSON.parse_string(columns[3])
             }

     func can_unlock(id: String, currency: int) -> bool:
         if owned_nodes.has(id):
             return false
         var node := nodes.get(id)
         if not node:
             return false
         for req in node["prereq"]:
             if req != "" and not owned_nodes.has(req):
                 return false
         return currency >= node["cost"]

     func unlock(id: String, currency: int) -> int:
         if not can_unlock(id, currency):
             return currency
         owned_nodes[id] = true
         return currency - nodes[id]["cost"]
     ```
4. **UI**
   - `scenes/ui/TechTreeScreen.tscn` : `Control` avec `GraphEdit` + `GraphNode` pour chaque tech. Positionne via `columns` (utilise CSV). 
   - Bouton `Unlock` appelle `TechTreeManager.unlock`. Montre un toast `Unlocked X`.
5. **Effets**
   - Connecte `InventoryBus.item_picked_up` pour ajouter `tech_currency`.
   - Lorsqu'un node `slot_extra` est débloqué, augmente la taille de `CargoGrid` (appelle `resize(7,4)`).

### Tests & validations
- `test_techtree_unlock_prereq.gd` : impossible de débloquer sans prérequis.
- `test_techtree_effect_applies.gd` : vérifier qu'un node `fuel_regen` augmente `ResourceManager`.
- Capture d'écran `docs/meta/tech_tree_screen.png`.

### Critères d'acceptation
- Le tech tree affiche au moins 3 niveaux de profondeur.
- Les coûts sont stockés dans un `int` (pas de string).

## Sprint 3 — Atlas de campagne persistant

### Résultat attendu
Une carte globale 12 nœuds persistant entre les runs, impactant les modificateurs de mission.

### Checklist
1. **Data**
   - `game/data/atlas.json` :
     ```json
     {
       "nodes": [
         {"id": "sector_alpha", "neighbors": ["sector_beta"], "modifier": "mud_everywhere"},
         {"id": "sector_beta", "neighbors": ["sector_alpha", "sector_gamma"], "modifier": "steel_floor"}
       ],
       "factions": ["otters", "krakens"],
       "ownership": {"sector_alpha": "otters", "sector_beta": "krakens"}
     }
     ```
2. **AtlasManager**
   - `scripts/meta/AtlasManager.gd` : charge JSON, expose `capture_node(node_id: String, faction: String)` et met à jour `MetaState.atlas_state`.
   - Connecte `EventBus` : quand un objectif est réussi, capture le nœud courant.
3. **UI Warfront**
   - `scenes/ui/WarfrontMap.tscn` : `TextureRect` + `Button` pour chaque node. Clique = lance une nouvelle run avec `LevelBlueprint` modifié (modif appliqué via `LevelGenerator`).
4. **Persistence**
   - Lors de `MetaSerializer.save_state`, inclure `AtlasManager.serialize()`.

### Tests & validations
- `test_atlas_capture_updates_state.gd` : capture `sector_beta` → `ownership` devient `otters`.
- `test_atlas_persists_after_reload.gd` : sauve, recharge, vérifie.
- Capture `docs/meta/atlas_map.png`.

### Critères d'acceptation
- Minimum 12 nœuds définis.
- Chaque nœud a au moins 2 voisins (sauf extrémités).

## Sprint 4 — Analytics & .env

### Résultat attendu
Collecte de métriques de run (durée, kills, loot) exportées en JSON + configuration de la clé AES via `.env`.

### Checklist
1. **RunAnalytics**
   - `scripts/meta/RunAnalytics.gd` : `start_run()`, `end_run(result: String)` enregistrant timestamp, ennemis tués (`Telemetry`), loot collecté (`InventoryBus`). Sauvegarde `user://analytics/run_<timestamp>.json`.
2. **.env gestion clé**
   - Crée `.env.example` avec `META_SECRET_KEY=remplace_moi`. Ajoute `.env` à `.gitignore`.
   - Modifie `MetaSerializer` pour lire la clé :
     ```gdscript
     var key := OS.get_environment("META_SECRET_KEY", SECRET_KEY)
     var encrypted := Encryptor.encrypt_aes256(data.to_utf8_buffer(), key)
     ```
3. **CI**
   - Ajoute `META_SECRET_KEY` dans GitHub Secrets.
4. **Tests**
   - `test_run_analytics_file_created.gd` : appelle `start_run`, `end_run`, vérifie le fichier.
   - `test_env_key_override.gd` : set env var dans test (utilise `OS.set_environment`).
5. **Documentation**
   - Ajoute `docs/meta/analytics.md` listant les champs collectés.

### Critères d'acceptation
- Chaque run crée un fichier <= 50 Ko.
- `.env.example` documenté dans README.

## Dépendances
- Nécessite loot (E5) pour currency, objectives (E4) pour atlas progression.
- Fournit données au Director (E7) et Endgame (E8).

## Risques & mitigations
- **Perte de données** : double-save (écrire sur `meta.save.tmp` puis renommer).
- **Clé exposée** : jamais commit `.env` (CI vérifie). Ajoute job `git-secret-check` dans pipeline.
- **Atlas déséquilibré** : log capture rate (Telemetry `atlas_captures`).
