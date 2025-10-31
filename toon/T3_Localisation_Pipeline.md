---
id: T3
name: "T3 — Pipeline de localisation & i18n"
kind: ticket
status: todo
epic: E5
owners:
  - engineering
  - localisation
estimate: 7j
risks:
  - Chaîne non extraite provoquant une régression UX
  - Polices fallback manquantes pour CJK
  - Coupes de texte dans l'UI non détectées avant build
---

# T3 — Pipeline de localisation & i18n

## Résumé
Mettre en place un pipeline complet d'extraction des chaînes, traduction FR/EN et fallback typographique pour préparer l'ouverture à d'autres langues sans régressions UI.

## Pré-requis exacts
- Installer `gettext` et `polib`.
- Définir la charte linguistique FR/EN avec glossaire partagé.
- Collecter les polices open-source avec support Latin + CJK (Noto Sans).

## Tâches détaillées
1. **Extraction des chaînes**
   - Annoter les scripts GDScript avec `tr()`.
   - Créer `tools/extract_strings.py` qui génère des fichiers `.po` dans `locales/<lang>/game.po`.
   - Ajouter un job CI `localisation-check` qui échoue si des chaînes non traduites sont détectées.
2. **Gestion des assets UI**
   - Centraliser les textes UI dans `data/strings/ui.csv` pour les éléments hors script.
   - Intégrer un loader `scripts/core/localization_manager.gd` capable de charger CSV + PO.
3. **Fonts & fallback**
   - Configurer `themes/DefaultTheme.tres` avec fallback Noto Sans CJK, Noto Sans Symbols.
   - Vérifier les glyphes manquants via `tools/check_font_coverage.py`.
4. **Mesure de troncation**
   - Ajouter un mode debug "UI Overflow" qui colorise les labels dépassant leur rect.
   - Générer un rapport via `godot --headless --path ./game --run ui_overflow_report` pour EN/FR.
5. **Intégration runtime**
   - Ajouter un menu de sélection de langue.
   - Persister la langue choisie et recharger les ressources dynamiquement.

## Tests & validations obligatoires
- `python tools/extract_strings.py --check`
- `gdunit4 -s tests/core/test_localization_manager.gd`
- Rapport `ui_overflow_report` sans dépassement pour EN/FR.

## Definition of Done
- Bascule FR/EN en jeu sans redémarrage.
- Scan TOON ne signale aucune chaîne orpheline.
- Aucune troncation ou overlap détecté sur EN/FR.
- Documentation `docs/localisation/pipeline.md` décrivant l'utilisation des scripts.

## Livrables / Artefacts
- `tools/extract_strings.py`
- `tools/check_font_coverage.py`
- `scripts/core/localization_manager.gd`
- `docs/localisation/pipeline.md`
