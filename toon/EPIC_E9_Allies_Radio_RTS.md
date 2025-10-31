---
id: E9
name: "EPIC E9 — Alliés Radio RTS"
kind: epic
status: todo
owners:
  - engineering
summary: "Ajouter les escouades alliées contrôlées par radio, leurs ordres diegetic, et l'intégration avec la ligne de front."
dependencies:
  - E1
  - E2
  - E3
  - E4
  - E5
  - E6
  - E7
  - E8
critical_path: false
---

# EPIC E9 — Alliés Radio RTS

## Objectif pédagogique
Même si cet epic n'est pas sur le chemin critique, il doit être traité avec la même rigueur : tu ajoutes des alliés commandés via radio, leur IA simple et la ligne de front dynamique.

## Préparation commune
- Rédige `docs/allies/order_voice_lines.txt` (liste des répliques radio, FR + EN).
- Télécharge SFX radio (freesound `RadioSquawk.wav`) et place dans `game/audio/radio/`.
- Ajoute un autoload `scripts/allies/AlliesBus.gd` (copie-colle gabarit plus bas).

## Sprint 1 — Types d'unités alliées

### Résultat attendu
Trois types d'unités avec comportements distincts : drones (follow), sapeurs (pose mines), half-tracks (tir suppressif).

### Checklist
1. **AlliesBus**
   - `scripts/allies/AlliesBus.gd` :
     ```gdscript
     extends Node

     signal order_issued(order: String, payload: Dictionary)
     signal unit_spawned(unit_type: String)

     func emit_order(order: String, payload: Dictionary = {}) -> void:
         emit_signal("order_issued", order, payload)

     func notify_spawn(unit_type: String) -> void:
         emit_signal("unit_spawned", unit_type)
     ```
   - Ajoute en autoload.
2. **Unités**
   - `scenes/allies/Drone.tscn` (CharacterBody2D) : suit le tank, tire faiblement.
   - `scenes/allies/Sapper.tscn` : pose `Mine.tscn` toutes les 15 s.
   - `scenes/allies/HalfTrack.tscn` : `VehicleBody2D` avec mitrailleuse.
   - Scripts correspondants (`scripts/allies/Drone.gd`, etc.) : assure-toi de loguer `AlliesBus.notify_spawn("drone")`.
3. **Pooling**
   - Utilise `ObjectPool` (E1) pour gérer les unités.
4. **Tests**
   - `test_drone_follows_tank.gd` : simulate path -> distance < 200 px.
   - `test_sapper_places_mines.gd` : après 30 s, 2 mines au sol.
   - `test_halftrack_fire_rate.gd` : check `EventBus` `ALLY_FIRE`.

### Critères d'acceptation
- Chaque unité a une icône HUD (CanvasLayer `AlliesPanel.tscn`).
- Les scripts ne contiennent aucun TODO.

## Sprint 2 — Système d'ordres radio

### Résultat attendu
Une interface radio circulaire propose 4 ordres : `Follow`, `Hold`, `Harass`, `EscortConvoy`. Les ordres s'exécutent via `AlliesBus`.

### Checklist
1. **UI Radio**
   - `scenes/ui/RadioMenu.tscn` : `CanvasLayer` avec 4 boutons (gauche/droite/haut/bas). Navigable au pad.
   - Script `scripts/ui/RadioMenu.gd` :
     ```gdscript
     extends CanvasLayer

     var _current_order := "follow"

     func show_menu() -> void:
         visible = true
         $AnimationPlayer.play("open")

     func hide_menu() -> void:
         $AnimationPlayer.play("close")
         await $AnimationPlayer.animation_finished
         visible = false

     func confirm_order() -> void:
         AlliesBus.emit_order(_current_order, {"timestamp": Time.get_unix_time_from_system()})
         _play_voice_line(_current_order)
     ```
   - `_play_voice_line` lit `order_voice_lines.txt` et joue un SFX.
2. **Input**
   - Ajoute action `radio_menu` (touche `R`). Appui long (>0.2 s) ouvre le menu, relâchement confirme.
3. **Ordres**
   - Implémente dans chaque unité une méthode `_apply_order(order: String, payload: Dictionary)`.
   - `Follow` → unité suit le tank.
   - `Hold` → stop, augmente `armor`.
   - `Harass` → se déplace vers la source d'ennemis la plus proche (`InfluenceGrid` E3).
   - `EscortConvoy` → suit le convoi actif (E2 Sprint 4).
   - Connecte `AlliesBus.order_issued` dans chaque script.
4. **Tests**
   - `test_radio_order_follow.gd` : après ordre, Drone distance < 100 px.
   - `test_radio_order_hold.gd` : vitesse = 0.
   - `test_radio_menu_input.gd` : simule appui long `R` -> menu visible.

### Critères d'acceptation
- Les ordres sont loggés dans `Telemetry.increment("orders_follow")` etc.
- La voix radio se joue (vérifie log `AudioServer.get_bus_peak_volume_db`).

## Sprint 3 — Ligne de front & influence

### Résultat attendu
Une ligne de front se déplace selon les actions du joueur et des alliés, donnant bonus quand elle avance.

### Checklist
1. **FrontLineManager**
   - `scripts/allies/FrontLineManager.gd` :
     ```gdscript
     extends Node

     signal frontline_shifted(value: float)

     var _position: float = 0.0

     func apply_progress(amount: float) -> void:
         _position = clamp(_position + amount, -100.0, 100.0)
         emit_signal("frontline_shifted", _position)
         Telemetry.set_gauge("frontline", _position)
     ```
   - Ajoute autoload.
2. **Contribution**
   - Chaque ennemi tué par une unité alliée → `apply_progress(+1)`. Si convoi détruit → `apply_progress(-5)`.
   - Récupère les événements via `EventBus`.
3. **Bonus**
   - Quand `_position > 50`, augmente drop rate (`DropManager`). Quand `< -50`, spawn plus d'ennemis (`EnemySpawner`).
4. **UI**
   - `scenes/ui/FrontlineBar.tscn` : barre horizontale montrant la progression. Couleurs : bleu (alliés) ↔ rouge (ennemi).
5. **Tests**
   - `test_frontline_progress_clamped.gd` : ne dépasse pas 100.
   - `test_frontline_bonus_applied.gd` : position > 50 → `DropManager` appelé.

### Critères d'acceptation
- Frontline mise à jour toutes les 2 s (Timer). Pas de jitter.
- UI reflète la valeur (0 = centre).

## Sprint 4 — Synergie Director & Endgame

### Résultat attendu
Les alliés interagissent avec le Director (E7) et les modes Endgame (E8). Les ordres modifient la Threat Clock et les récompenses.

### Checklist
1. **Director hooks**
   - Quand un ordre `Harass` est actif, `DirectorController` réduit la Threat Clock de 5 toutes les 30 s.
   - Si `Hold` > 20 s, Threat Clock augmente de 3 (moins d'action).
2. **Endgame integration**
   - Dans Opérations (E8), autorise un slot d'allié supplémentaire (Key reward). Documente dans `docs/endgame/endgame_design.md`.
   - Dans Rifts, chaque palier atteint + `apply_progress(+2)` à la frontline globale.
3. **Rewards**
   - Ajoute drop `ally_upgrade_token` (items.json). Permet d'acheter upgrade (HP, dégâts) via `TechTreeManager`.
4. **Tests**
   - `test_harass_reduces_threat.gd` : active harass, avance temps 30 s → Threat -5.
   - `test_hold_increases_threat.gd` : hold 25 s → Threat +3.
   - `test_ally_upgrade_token_available.gd` : drop -> TechTree node débloqué.
5. **Documentation**
   - Ajoute `docs/allies/ally_upgrade_flow.md` (schéma).

### Critères d'acceptation
- Aucun ordre n'est ignoré (log `AlliesBus` audit `docs/allies/orders_log.csv`).
- Threat Clock reste dans [0, 100].

## Dépendances
- S'appuie sur Director (E7) et Endgame (E8). Fournit un enrichissement optionnel.

## Risques & mitigations
- **Surcharge UI** : possibilité de masquer le RadioMenu via options (`MetaState.options`).
- **Pathfinding alliés** : utiliser `NavigationServer2D` avec couche dédiée.
- **Audio répétitif** : cycle les voice lines (ne pas jouer la même 2 fois d'affilée, stocker `_last_line`).
