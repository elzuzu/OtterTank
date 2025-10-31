---
id: T7
name: "T7 — Budgets performance & mémoire"
kind: ticket
status: todo
epic: E4
owners:
  - engineering
  - tech_art
  - qa
estimate: 6j
risks:
  - Budgets irréalistes vs matériel cible
  - Surprofiling qui ralentit la prod
  - Oubli de mettre à jour les limites après optimisation
  - Optimisations qui compromettent le feel KBM (accélération implicite, smoothing)
---

# T7 — Budgets performance & mémoire

## Résumé
Définir et automatiser les budgets CPU, GPU, VRAM et densité d'entités pour garantir 60 fps sur machine médiane et éviter les dérives de contenu.

## Pré-requis exacts
- Importer les budgets frame pacing de T1 — KBM Gold Standard (p95 ≤16.6 ms) comme contraintes incontournables.
- Matériel cible défini (CPU Ryzen 5 3600, GPU GTX 1060, 16 Go RAM).
- Accès à `Godot Performance Profiler` et `pixi` pour profiling batch.
- Connaître la taille moyenne des assets actuels.

## Tâches détaillées
1. **Définition des budgets**
   - Fixer p95 CPU ≤16.6 ms, GPU ≤16.6 ms, VRAM ≤3.5 Go, RAM ≤8 Go.
   - Limiter densité mobs/projos par chunk (ex: 30 mobs actifs, 80 projectiles).
   - Documenter dans `docs/perf/budgets.md`.
2. **Instrumentation**
   - Ajouter des compteurs runtime (nombre d'entités, effets VFX) avec alertes debug.
   - Intégrer `PerformanceMonitor` dans `autoload/Telemetry.gd`.
3. **Tests automatisés**
   - Créer un scénario `res://tests/perf/baseline_battle.tscn` (10 min) rejouable.
   - Script `tools/run_perf_profile.py` capturant CPU/GPU/VRAM p95.
   - Faille pipeline si dépassement budgets.
4. **CI & rapports**
   - Ajouter job `perf-budget` qui exécute le scénario sur machine dédiée.
   - Générer rapports HTML dans `docs/perf/reports/<build>.html`.

## Tests & validations obligatoires
- `python tools/run_perf_profile.py --scenario baseline_battle --duration 600`
- Revue QA des rapports sur 3 builds consécutifs.
- Validation tech art sur VRAM (atlas ≤ budget).

## Definition of Done
- Budgets documentés et approuvés.
- Pipeline de test perf échoue au-delà des limites.
- Rapport perf pour la build courante archivé.
- Densité mobs/projos clampée runtime.

## Livrables / Artefacts
- `tools/run_perf_profile.py`
- `docs/perf/budgets.md`
- `docs/perf/reports/`
- `tests/perf/baseline_battle.tscn`
