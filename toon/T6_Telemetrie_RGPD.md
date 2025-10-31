---
id: T6
name: "T6 — Télémétrie respectueuse & conformité RGPD"
kind: ticket
status: todo
epic: E6
owners:
  - engineering
  - data
  - legal
estimate: 7j
risks:
  - Collecte accidentelle de données personnelles
  - Refus de consentement bloquant les statistiques essentielles
  - Incohérence entre doc légale et implémentation
---

# T6 — Télémétrie respectueuse & conformité RGPD

## Résumé
Mettre en place une télémétrie opt-in, anonymisée et documentée pour suivre les métriques de gameplay clés tout en respectant strictement la réglementation RGPD (Europe/Zurich).

## Pré-requis exacts
- Consultation juridique confirmant la liste de métriques autorisées.
- Stockage télémetrie défini (BigQuery ou fichiers locaux anonymisés).
- Préparer `PRIVACY.md` modèle validé.

## Tâches détaillées
1. **Consentement**
   - Ajouter un écran initial explicitant la collecte, avec options Accepter/Refuser.
   - Stocker le choix dans `user://config/privacy.json`.
   - Permettre de reconsentir via le menu Options.
2. **Collecte anonymisée**
   - Générer un identifiant aléatoire (UUIDv4) stocké localement et non exporté.
   - Collecter les métriques : durée de run, DPS moyen, FPS moyen, taux de victoire, distribution des drops, décès par cause.
   - Ne collecter aucune donnée perso (nom, email, IP).
3. **Pipeline & export**
   - Mettre en place un buffer local `telemetry/session_*.jsonl`.
   - Ajouter un script `tools/export_telemetry_csv.py` pour agrégation locale.
   - Ajouter un job CI qui anonymise encore (suppression UUID) avant upload éventuel.
4. **Documentation & conformité**
   - Rédiger `PRIVACY.md` détaillant les métriques, la base légale, la durée de conservation et l'opt-out.
   - Tenir un registre de traitement (modèle simple) dans `docs/privacy/registre_traitement.md`.

## Tests & validations obligatoires
- `gdunit4 -s tests/telemetry/test_consent_flow.gd`
- Simulation d'une session avec opt-in, vérification des métriques enregistrées.
- Revue légale signée et jointe.

## Definition of Done
- Consentement explicite requis et stocké.
- Télémetry désactivée automatiquement si opt-out.
- Export CSV local fonctionnel sans PII.
- Documentation PRIVACY.md à jour et revue juridique attachée.

## Livrables / Artefacts
- `autoload/Telemetry.gd`
- `tools/export_telemetry_csv.py`
- `PRIVACY.md`
- `docs/privacy/registre_traitement.md`
