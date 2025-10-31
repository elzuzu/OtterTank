---
id: T2
name: "T2 — Accessibilité & confort visuel"
kind: ticket
status: todo
epic: E2
owners:
  - ux
  - engineering
  - qa
estimate: 6j
risks:
  - Perte de lisibilité des effets lors de l'application de LUT
  - Sous-titres radio qui masquent le HUD critique
  - Paramètres non persistés entraînant des régressions UX
  - Accessibilité qui désactive par erreur les réglages du KBM Gold Standard
---

# T2 — Accessibilité & confort visuel

## Résumé
Mettre en place les options d'accessibilité critiques (modes daltonisme, réduction des mouvements, sous-titres radio, HUD lisible) afin d'élargir l'audience et réduire la fatigue visuelle et le motion sickness.

## Pré-requis exacts
- T1 — KBM Gold Standard (P0) validé et verrouillé (tous les toggles d'options doivent rester compatibles).
- Liste d'effets visuels sensibles (flash, secousses, bloom).
- Accessibilité guidelines WCAG 2.1 AA en référence.
- Panneau QA interne avec 5 testeurs volontaires.

## Tâches détaillées
1. **Options visuelles**
   - Ajouter un toggle "Réduction des mouvements" qui désactive caméra shake, motion blur et réduit la fréquence des particules.
   - Créer un profil "Éclair minimal" réduisant intensité des flashs (shader `materials/vfx_flash.tres`).
2. **Modes daltonisme**
   - Générer 3 LUTs (deuteranopie, protanopie, tritanopie) stockées dans `assets/luts/`.
   - Implémenter le post-process LUT dans `scenes/core/post_process.tscn` avec blend ajustable.
   - Ajouter une prévisualisation directe dans le menu Options.
3. **Sous-titres radio**
   - Ajouter `scenes/ui/subtitles_radio.tscn` avec options de taille (S/M/L), fond semi-transparent et indication de locuteur.
   - Synchroniser avec les événements audio via `scripts/systems/radio_manager.gd`.
4. **HUD lisibilité**
   - Intégrer un slider "Taille HUD" (100–150%).
   - Ajouter un curseur "Lisibilité loot" qui active outline + glow sur items rares.
   - Vérifier le contraste des éléments HUD (ratio ≥ 4.5:1).
5. **Persistance et télémétrie**
   - Sauvegarder les réglages dans `GameConfig`.
   - Logguer l'activation des options (sans données personnelles) pour mesurer l'usage.

## Tests & validations obligatoires
- `gdunit4 -s tests/ui/test_accessibility_options.gd`
- Rapport QA : 5 testeurs, ≥80% jugent l'écran clair.
- Check contraste via script `tools/check_contrast.py`.

## Definition of Done
- Tous les réglages accessibles depuis le menu Options > Accessibilité.
- Options appliquées en temps réel et persistées sur relance.
- Sous-titres radio couvrent 100% des messages vocaux existants.
- Rapport QA validé et archivé dans `docs/accessibility/qa_session.md`.

## Livrables / Artefacts
- `scenes/ui/options_accessibility.tscn`
- `scenes/ui/subtitles_radio.tscn`
- `scripts/systems/radio_manager.gd`
- `tools/check_contrast.py`
- `docs/accessibility/qa_session.md`
