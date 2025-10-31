---
id: T1
name: "T1 — KBM Gold Standard (P0)"
kind: ticket
status: todo
epic: E1
owners:
  - engineering
  - ux
  - qa
estimate: 9j
risks:
  - Régression de latence ou de sensibilité ressentie par rapport aux builds antérieurs
  - Difficulté à concilier camera shake et lisibilité loot/HUD
  - Budget perf insuffisant pour maintenir le tick fixe à 60 FPS pendant les stress tests
---

# T1 — KBM Gold Standard (P0)

## Résumé
Offrir une expérience clavier/souris chirurgicale pour un gameplay "Cannon Fodder-like" : souris raw, turret-first aiming, mouvements nerveux et caméra guidée par le curseur, avec toutes les options d'accessibilité pertinentes. La manette est explicitement reléguée en P2 — cette tâche verrouille le feel KBM qui servira de référence à tout le reste du backlog.

## Pré-requis exacts
- Godot 4.1.3 LTS et profil Input correctement configuré.
- Accès à un PC 60 FPS stable (moniteur 60/120 Hz) pour mesurer la latence perçue.
- Activation de l'overlay debug `input_overlay.tscn` pour visualiser tick, frame pacing et diff angle.
- Lecture préalable des guidelines d'accessibilité (T2) pour assurer les toggles caméra/shake.

## Tâches détaillées
1. **Souris raw & sensibilité linéaire**
   - Lire les deltas souris dans `_input(event)` avec `InputEventMouseMotion.relative`.
   - Ajouter `input/raw_mouse_mode = true` dans `project.godot` et exposer un toggle "Cursor focus" (fort/faible) dans `settings/input_kbm.tres`.
   - Implémenter une courbe de sensibilité strictement linéaire (0.1 → 3.5) via `InputMapper` ; fournir un preset optionnel "courbe douce".
   - Veiller à ce que toute accélération soit désactivée par défaut.
2. **Turret-first aiming & lock**
   - Faire suivre la tourelle au curseur écran (`get_viewport().get_mouse_position()`) converti en monde via `Camera2D`.
   - Ajouter un paramètre de snap (0–8 ms) dans `tank/weapon_controller.gd` avec interpolation contrôlée.
   - Implémenter `Space` en "lock tourelle" pour strafe sans modifier l'angle de tir ; bufferiser l'input (80–120 ms).
3. **Mouvement nerveux lisible**
   - Configurer friction anisotrope : inertie latérale > frontale dans `tank/chassis_controller.gd`.
   - Ajouter un `drift_assist` léger et un dash/boost sur `Shift` avec buffer input.
   - Vérifier que la trajectoire reste lisible (trails, skid marks optionnels, pas de blur).
4. **Caméra mouse-led**
   - Implémenter une soft-zone centrée sur le tank avec offset vers le curseur (faible) et zoom dynamique à haute vitesse.
   - Ajouter un shake court (<=120 ms) sur impacts majeurs avec toggle "réduction mouvements" dans les options.
5. **HUD & visée**
   - Créer un réticule avec indicateur de lead optionnel pour projectiles lents.
   - Ajouter un crosshair coloré lorsqu'une cible prioritaire entre dans un cône configurable (sans aim assist caché).
6. **Bindings par défaut**
   - Définir `ZQSD/WASD` déplacement, `LMB` tir principal, `RMB` alt-fire, roulette pour cycler les munitions.
   - Affecter `1–4` pour la sélection directe, `F` salvage/ramassage, `Tab` inventaire rapide (radial optionnel).
7. **Performance & frame pacing**
   - Verrouiller le tick gameplay à 60 Hz, lire les inputs chaque frame.
   - Mettre en place un overlay debug affichant p95 frame time, tick budget et backlog input.
   - Garantir p95 ≤ 16.6 ms lors du stress test standard E1.
8. **Options rapides en jeu**
   - Offrir en pause un panneau "Options KBM" : sensibilité, inversion X/Y, indicator de lead, cursor focus, toggles shake/flash.
   - Persister ces options dans `user://config/input_kbm.json`.

## Tests & validations obligatoires
- `godot --headless --path ./game --run test_input_latency`
- `gdunit4 -s tests/input/test_mouse_curve.gd`
- Session QA interne 5 testeurs : feedback latence ≥4/5 (compte rendu consigné).
- Mesure "test flick" : à 60 FPS, écart angle cible <2° sur 10 essais (journalisé dans `docs/playtests/kbm_gold_standard.md`).
- "Strafe test" : lock tourelle réduit la dispersion des tirs de ≥30% (rapport QA comparatif).

## Definition of Done
- Tous les bindings KBM listés fonctionnent et sont persistés.
- Les options caméra/flash/shake peuvent être désactivées sans perte de lisibilité du loot.
- L'overlay debug montre p95 ≤16.6 ms et tick stable à 60 Hz pendant 10 minutes de stress test.
- Les critères d'acceptation (flick, strafe, latence perçue, accessibilité) sont validés et archivés.
- Documentation mise à jour dans `docs/controls/kbm.md` avec captures des options.

## Livrables / Artefacts
- `scripts/core/input_manager.gd`
- `tank/chassis_controller.gd`
- `tank/weapon_controller.gd`
- `scenes/ui/options_kbm.tscn`
- `docs/controls/kbm.md`
- `docs/playtests/kbm_gold_standard.md`
