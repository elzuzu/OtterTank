---
id: T1
name: "T1 — Support manette complet + Steam Deck"
kind: ticket
status: todo
epic: E1
owners:
  - engineering
  - ux
  - qa
estimate: 8j
risks:
  - Drift sur la calibration des sticks ou rumble trop agressif
  - Incompatibilité Steam Input / Godot InputMap
  - Décalage HUD/controller qui casse l'accessibilité
---

# T1 — Support manette complet + Steam Deck

## Résumé
Garantir un support twin-stick impeccable pour XInput/SDL et Steam Deck, incluant les vibrations, deadzones adaptatives, aim assist réglable et un mapping UI complet, afin que 100% du jeu soit jouable à la manette sans clavier.

## Pré-requis exacts
- Godot 4.1.3 LTS avec module Input remappé.
- Accès à une manette Xbox Series, DualSense et à un Steam Deck pour validation.
- Installer `steam-runtime` pour tester le mode Deck Remote.
- Activer l'overlay Steam Input en mode développeur.

## Tâches détaillées
1. **Cartographie & profils**
   - Définir les profils XInput, SDL Standard et DualSense dans `game/input/controller_profiles.json`.
   - Ajouter l'autodétection à chaud dans `scripts/core/input_manager.gd` (signal `joy_connection_changed`).
   - Mapper toutes les actions InputMap (mouvement, visée, tir, dash, interactions, UI navigation).
2. **Deadzones & courbes**
   - Implémenter des deadzones dynamiques (linéaire + exponentielle) et exposer les paramètres dans `settings/controller.tres`.
   - Ajouter un test gdUnit4 qui vérifie que la deadzone minimale n'excède pas 0.12 et que la réponse reste monotone.
3. **Aim assist**
   - Ajouter un module `scripts/systems/aim_assist.gd` avec modes Off / Léger / Fort.
   - Créer des gizmos debug pour visualiser le cône d'aide dans l'éditeur (`DebugCanvas`).
   - Relier les paramètres au menu Options (`scenes/ui/options_controller.tscn`).
4. **Rumble & feedback**
   - Implémenter les vibrations contextuelles (impact projectile, dégâts reçus, dash réussi) via `Input.start_joy_vibration()`.
   - Prévoir un fallback pour manettes sans rumble (log `Telemetry`).
   - Ajouter un toggle "Vibrations" dans les options.
5. **Steam Deck**
   - Créer un preset Deck (layout par défaut, gyroscope Off) chargé automatiquement en détectant `OS.has_feature("steam_deck")`.
   - Vérifier les overlays d'aide (prompts ABXY) dans `scenes/ui/hud_prompts.tscn`.
   - Documenter la procédure de test dans `docs/controls/steam_deck.md`.
6. **Persistance**
   - Sérialiser les paramètres contrôleur dans `user://config/controller.json` via `GameConfig`.
   - Ajouter un test d'intégration qui sauvegarde/modifie/recharge ces paramètres.

## Tests & validations obligatoires
- `gdunit4 -s tests/input/test_controller_profiles.gd`
- `godot --headless --path ./game --run test_controller_settings`
- Session Steam Deck de 30 minutes sans retour bloquant (journal QA).

## Definition of Done
- Manette reconnue à chaud, aucun redémarrage requis.
- Toutes les actions du jeu accessibles sans clavier/souris.
- Options de sensibilité, aim assist et vibrations persistées après redémarrage.
- Validation QA Steam Deck (rapport signé) sans blocants.
- Documentation `docs/controls/steam_deck.md` mise à jour avec captures.

## Livrables / Artefacts
- `game/input/controller_profiles.json`
- `scripts/core/input_manager.gd`
- `scenes/ui/options_controller.tscn`
- `docs/controls/steam_deck.md`
