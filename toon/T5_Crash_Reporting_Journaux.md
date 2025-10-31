---
id: T5
name: "T5 — Crash reporting & journaux corrélés"
kind: ticket
status: todo
epic: E1
owners:
  - engineering
  - qa
estimate: 5j
risks:
  - Envoi de données personnelles non souhaité
  - Crash handler qui dégrade les performances
  - Logs verbeux saturant le disque
---

# T5 — Crash reporting & journaux corrélés

## Résumé
Intégrer une solution de crash reporting (Sentry ou équivalent) avec logs corrélés aux seeds/builds afin de reproduire rapidement les incidents liés au procedural.

## Pré-requis exacts
- Créer un compte Sentry projet "OtterTank" (environnement dev/staging/prod).
- Générer DSN et tokens stockés dans `.env` (non commité).
- Ajouter `godot-sentry` au projet.

## Tâches détaillées
1. **Instrumentation**
   - Configurer `autoload/CrashReporter.gd` pour init Sentry au démarrage.
   - Capturer les seeds, build hash, config joueur dans chaque événement.
   - Ajouter un fallback local `logs/crash_reports/` si Sentry indisponible.
2. **Logging structuré**
   - Mettre en place `scripts/core/logger.gd` (format JSON lines) avec rotation.
   - Logguer les seeds générées, la Threat Clock, les spawns majeurs.
3. **Crash volontaire**
   - Créer une commande debug `--force-crash` pour tester la pipeline.
   - Vérifier que l'événement apparaît dans Sentry avec contexte complet.
4. **CI & artifacts**
   - Ajouter l'ID du build CI dans les logs (`CI_COMMIT_SHA`).
   - Archiver `logs/session_*.jsonl` dans les artefacts CI pour 24h.

## Tests & validations obligatoires
- `godot --headless --path ./game -- --force-crash`
- Vérifier dans Sentry l'événement (capture écran jointe dans ticket).
- Vérifier la présence des logs corrélés dans `logs/`.

## Definition of Done
- Crash volontaire remonte seed/build/config dans Sentry.
- Aucun PII transmis (audit log).
- Logs accessibles depuis CI et rotation automatique fonctionnelle.
- Documentation `docs/observability/crash_reporting.md` avec procédures.

## Livrables / Artefacts
- `autoload/CrashReporter.gd`
- `scripts/core/logger.gd`
- `docs/observability/crash_reporting.md`
