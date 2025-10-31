---
id: E7
name: "EPIC E7 — Director Adaptatif & Threat Clock"
kind: epic
status: todo
owners:
  - engineering
summary: "Créer le Director qui observe le build du joueur, ajuste les vagues ennemies, déclenche des contre-technologies et gère la Threat Clock."
dependencies:
  - E1
  - E2
  - E3
  - E4
  - E5
  - E6
critical_path: true
---

# EPIC E7 — Director Adaptatif & Threat Clock

## Objectif pédagogique
Tu dois coder un Director dynamique comme dans Left4Dead mais pour OtterTank : il surveille le build, modifie les spawns, déclenche des contre-technos, et gère une horloge de menace. Chaque sprint détaille les scripts et tests.

## Préparation commune
- Relis `T1 — KBM Gold Standard` : le Director ne doit jamais appliquer de handicaps modifiant la latence input ou la courbe souris, uniquement des menaces gameplay.
- Lire la doc Valve "AI Director" (résumé dans `docs/director/ai_director_summary.md` que tu rédiges).
- Créer `docs/director/metrics.xlsx` (colonnes : `Metric`, `Source`, `UpdateFrequency`, `Thresholds`).
- Ajouter un autoload `scripts/director/DirectorBus.gd` (vide pour l'instant).

## Sprint 1 — Collecte des métriques

### Résultat attendu
Un `BuildAnalyzer` récupère les statistiques du joueur (DPS, portée, AOE, mobilité) et les met à jour toutes les 5 s.

### Checklist
1. **BuildAnalyzer**
   - `scripts/director/BuildAnalyzer.gd` :
     ```gdscript
     extends Node

     class_name BuildAnalyzer

     signal metrics_updated(metrics: Dictionary)

     var _timer := Timer.new()

     func _ready() -> void:
         add_child(_timer)
         _timer.wait_time = 5.0
         _timer.timeout.connect(_refresh)
         _timer.start()

     func _refresh() -> void:
         var metrics := {
             "dps_burst": _compute_dps_burst(),
             "dps_sustain": _compute_dps_sustain(),
             "range": _compute_range(),
             "aoe_ratio": _compute_aoe_ratio(),
             "mobility": _compute_mobility()
         }
         emit_signal("metrics_updated", metrics)
         DirectorBus.emit_signal("build_metrics", metrics)

     func _compute_dps_burst() -> float:
         return InventoryBus.get_stat("burst_damage")

     func _compute_dps_sustain() -> float:
         return InventoryBus.get_stat("sustain_damage")

     func _compute_range() -> float:
         return InventoryBus.get_stat("weapon_range")

     func _compute_aoe_ratio() -> float:
         return InventoryBus.get_stat("aoe_ratio")

     func _compute_mobility() -> float:
         return ResourceManager.fuel
     ```
   - Ajoute des méthodes stub `InventoryBus.get_stat` (retourne 0 si absent).
2. **Telemetry**
   - Chaque mise à jour doit appeler `Telemetry.set_gauge("director_dps_burst", metrics["dps_burst"])` etc.
3. **Tests**
   - `test_build_analyzer_emits_metrics.gd` : stub `InventoryBus` pour renvoyer valeurs > 0 et vérifier signal.
   - `test_metrics_frequency.gd` : s'assurer que `_refresh` est appelé toutes les 5 s (utilise `advance_time`).

### Critères d'acceptation
- Fichier `docs/director/metrics.xlsx` rempli (4 métriques + description).
- `DirectorBus` expose le signal `build_metrics`.

## Sprint 2 — Threat Clock

### Résultat attendu
Une horloge de menace augmente sur la durée et déclenche des événements quand elle atteint 100.

### Checklist
1. **ThreatClock**
   - `scripts/director/ThreatClock.gd` :
     ```gdscript
     extends Node

     class_name ThreatClock

     signal threshold_reached(level: int)

     var value: float = 0.0
     var level: int = 0

     func add(amount: float) -> void:
         value = clamp(value + amount, 0.0, 100.0)
         if value >= 100.0:
             value = 0.0
             level += 1
             emit_signal("threshold_reached", level)
             DirectorBus.emit_signal("threat_level", level)
     ```
2. **Intégration**
   - Dans `Game.gd`, instancie `ThreatClock`, ajoute `ThreatClock.add(delta * 0.5)` chaque seconde.
   - Reset partiel : quand le joueur sauve un convoi (E2 Sprint 4), `ThreatClock.add(-20)`.
3. **UI**
   - Ajoute une jauge circulaire `ThreatClockWidget.tscn` (HUD). Couleur change (vert <33, orange <66, rouge sinon).
4. **Tests**
   - `test_threatclock_levels.gd` : ajoute 200 points → `level == 2`.
   - `test_threatclock_negative.gd` : assure que `add(-50)` ne passe pas sous 0.

### Critères d'acceptation
- Threat Clock visible sur HUD.
- Les resets négatifs fonctionnent.

## Sprint 3 — Contre-technologies adaptatives

### Résultat attendu
Toutes les 5 minutes, le Director sélectionne une contre-technologie basée sur les métriques et modifie le comportement ennemi.

### Checklist
1. **Contre-technos**
   - Crée `game/data/counter_technos.json` :
     ```json
     {
       "entries": [
         {"id": "anti_frag_shield", "metric": "aoe_ratio", "threshold": 0.5, "effect": "spawn_shield_units"},
         {"id": "emp_mines", "metric": "mobility", "threshold": 60, "effect": "deploy_emp_mines"}
       ]
     }
     ```
2. **DirectorController**
   - `scripts/director/DirectorController.gd` :
     ```gdscript
     extends Node

     class_name DirectorController

     @export var build_analyzer: BuildAnalyzer
     @export var threat_clock: ThreatClock

     var _entries := []
     var _rng := RandomNumberGenerator.new()
     var _timer := Timer.new()

     func _ready() -> void:
         _entries = _load_entries()
         add_child(_timer)
         _timer.wait_time = 300.0
         _timer.timeout.connect(_pick_counter_tech)
         _timer.start()
         DirectorBus.connect("build_metrics", Callable(self, "_on_build_metrics"))
         threat_clock.threshold_reached.connect(_on_threat_level)

     func _load_entries() -> Array:
         var file := FileAccess.open("res://data/counter_technos.json", FileAccess.READ)
         var data := JSON.parse_string(file.get_as_text())
         return data["entries"]

     var _last_metrics := {}

     func _on_build_metrics(metrics: Dictionary) -> void:
         _last_metrics = metrics

     func _pick_counter_tech() -> void:
         for entry in _entries:
             var metric_value := _last_metrics.get(entry["metric"], 0)
             if metric_value >= entry["threshold"]:
                 _apply_effect(entry["effect"])
                 EventBus.emit_event("COUNTER_TECH_TRIGGERED", entry)
                 return
         # fallback
         _apply_effect("deploy_sniper_wave")
         EventBus.emit_event("COUNTER_TECH_TRIGGERED", {"id": "fallback"})

     func _apply_effect(effect: String) -> void:
         match effect:
             "spawn_shield_units":
                 EnemySpawner.spawn_special("shield_bearer")
             "deploy_emp_mines":
                 EventBus.emit_event("SPAWN_EMP_MINES", {})
             "deploy_sniper_wave":
                 EnemySpawner.spawn_special("sniper")
     ```
3. **EnemySpawner**
   - Ajoute fonction `spawn_special(type: String)` qui instancie un enemy spécifique (E3 mini-boss, sniper, etc.).
4. **War Room**
   - Met à jour `WarRoomPanel` (E4) pour afficher les contre-technos déclenchées.

### Tests & validations
- `test_director_picks_matching_counter.gd` : metrics `aoe_ratio=0.6` → effect `spawn_shield_units`.
- `test_director_fallback.gd` : metrics bas → `deploy_sniper_wave`.
- Capture `docs/director/counter_tech_log.png` (War Room). 

### Critères d'acceptation
- Aucune contre-tech ne se déclenche deux fois de suite (utilise variable `_last_effect`).
- Les effets modifient réellement les ennemis (vérifie via log `EnemySpawner.spawn_special`).

## Sprint 4 — Threat Events & Audio dynamique

### Résultat attendu
À chaque palier de Threat, déclencher un événement (mini-boss, brouilleurs, etc.) et synchroniser l'audio "Ballistic Beats".

### Checklist
1. **Threat events**
   - Définis dans `data/threat_events.json` : level → `event_id`, `payload`. Exemple : `1: spawn_mini_boss`, `2: deploy_brouilleur`, `3: artillery_barrage`.
   - Dans `DirectorController._on_threat_level`, charge l'événement et l'applique (via `EventBus`).
2. **Audio synchronisation**
   - Ajoute `scripts/audio/BallisticBeats.gd` : écoute `EventBus` (`PROJECTILE_RICOCHET`, `OVERHEAT`). Utilise `AudioStreamGenerator` pour moduler un beat.
   - Connecte `DirectorBus.build_metrics` pour ajuster BPM (plus de DPS → BPM élevé).
3. **UI Feedback**
   - Ajoute `ThreatEventBanner.tscn` (HUD) affichant `Incoming: EMP Mines` pendant 3 s.
4. **Tests**
   - `test_threat_event_lookup.gd` : vérifie que le niveau 2 retourne `deploy_brouilleur`.
   - `test_ballistic_beats_changes_bpm.gd` : stub metrics, vérifier changement de BPM.
5. **Documentation**
   - Complète `docs/director/ai_director_summary.md` avec explication menace/audio.

### Critères d'acceptation
- Threat events ne déclenchent jamais plus d'un événement par niveau.
- Audio BPM loggé via `Telemetry.set_gauge("ballistic_bpm", bpm)`.

## Dépendances
- Utilise InfluenceGrid (E3), Objectives (E4), Loot stats (E5), Meta data (E6).
- Fournit modifications au Endgame (E8).

## Risques & mitigations
- **Surcharge audio** : limite les effets audio à 8 en simultané (`AudioServer.set_bus_effect_enabled`).
- **Boucle infinie** : `_pick_counter_tech` doit stocker l'heure du dernier déclenchement (`Time.get_ticks_msec()`).
- **Métriques nulles** : si `InventoryBus.get_stat` renvoie 0, applique une contre-tech neutre (`deploy_supply_drop`).
