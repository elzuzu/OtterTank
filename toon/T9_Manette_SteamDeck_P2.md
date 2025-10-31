---
id: T9
name: "T9 — Manette & Steam Deck (P2 optionnel)"
kind: ticket
status: backlog
epic: E1
owners:
  - engineering
  - ux
estimate: 4j
risks:
  - Risque de détourner des ressources du polish KBM si traité trop tôt
  - Divergence avec les presets Steam Input si non testés sur hardware réel
---

# T9 — Manette & Steam Deck (P2 optionnel)

## Résumé
Implémenter un support manette correct après la sortie KBM en reprenant les contrôles twin-stick, avec des deadzones configurables et un mapping complet, sans sacrifier l'équilibre ni le feel calibré dans T1. Ce ticket ne doit être ouvert qu'une fois le KBM Gold Standard validé et stabilisé.

## Pré-requis exacts
- T1 — KBM Gold Standard (P0) terminé et gelé (pas de régression d'input ouverte).
- Inventaire des actions finales dans `game/input/actions.json`.
- Accès à une manette XInput et à un Steam Deck pour validation.

## Tâches détaillées
1. **Mapping & profils**
   - Définir les profils XInput, SDL Standard et DualSense dans `game/input/controller_profiles.json`.
   - Implémenter la détection à chaud (`joy_connection_changed`) et charger le mapping correspondant.
2. **Deadzones & courbes**
   - Exposer les deadzones (linéaire / exponentielle) dans `settings/controller.tres` avec valeurs par défaut testées.
   - Ajouter un test gdUnit4 pour garantir une réponse monotone et une deadzone minimale ≤0.12.
3. **Feedback & options**
   - Ajouter un module de vibrations minimal (impact lourd, dégâts reçus, dash) avec toggle.
   - Étendre le menu Options avec un onglet "Manette" qui respecte les presets KBM existants.
4. **Steam Deck**
   - Créer un preset Steam Deck (layout, gyroscope off par défaut) et documenter la configuration dans `docs/controls/steam_deck.md`.
   - Vérifier la cohérence des prompts (ABXY) dans le HUD.

## Tests & validations obligatoires
- `gdunit4 -s tests/input/test_controller_profiles.gd`
- `godot --headless --path ./game --run test_controller_settings`
- Session QA 30 minutes sur Steam Deck ou Steam Input Desktop avec rapport.

## Definition of Done
- Toutes les actions principales jouables à la manette sans clavier.
- Options de sensibilité, deadzones et vibrations persistées et alignées avec T1.
- Documentation `docs/controls/steam_deck.md` mise à jour avec captures et presets.

## Livrables / Artefacts
- `game/input/controller_profiles.json`
- `scripts/core/input_manager.gd`
- `scenes/ui/options_controller.tscn`
- `docs/controls/steam_deck.md`
