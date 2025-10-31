# OtterTank — Tank Survivors Design Operating Manual (2025 Edition)

> **Objective**: ship a “Vampire Survivors meets Diablo with tanks” experience where terrain scars persist, the player kit is a modular war machine, and meta-systems keep players grinding for builds long after the campaign. This README distils the shared product vision, gameplay pillars, production plan, and tooling so any contributor or coding agent can onboard in minutes.

---

## 1. Product Overview

| Pillar | Description | Player Impact |
| --- | --- | --- |
| **Battlefield Sculpting** | Destructible terrain, carcasses, ricochets and craters that persist through the run (and, later, the warfront). | The arena becomes a tactical tool; players draw their escape routes with shells. |
| **Modular Field Kit** | Tanks are built from hot-swappable modules (châssis, tourelle, chenilles, moteur, utilitaires) that can be salvaged mid-run. | Constant agency: “pit-stop bubbles” let players cannibalise enemy tech for power spikes. |
| **Radio-RTS Escouades** | Drone wings, sapeurs et half-tracks obéissent à quatre ordres diegetic (Suivre, Tenir, Harceler, Escorter). | Bullet-heaven chaos with light tactical orchestration; the front line visibly shifts. |
| **Tri-Ressource Logistique** | Fuel, Heat et Ammo régissent mobilité, cadence et types de munitions; convois et drones cargo offrent des refills risqués. | Moment-to-moment tension: overdrive or conserve? Logistics events create memorable clutch plays. |
| **Director Adaptatif** | L’IA observe le build (AOE, portée, burst, mobilité) et déploie des contre-technos dynamiques que le joueur contre dans la War Room. | Chaque run reste fraîche : l’ennemi répond au build, forçant des choix renouvelés. |

### Game Structure Snapshot
- **Story Campaign**: acts procéduraux guidés (biomes Desert/Urbain/Forêt industrielle…) en 12–18 minutes, objectifs scénarisés (Escorter, Détruire, Tenir, Hacker) et boss gates.
- **Endgame Loop**: opérations à clés de 8–12 minutes, rifts chronométrés avec Threat Clock, warfront persistant par nœuds et World Tiers I–V débloquant de nouveaux modificateurs + iLvl.
- **Meta Progression**: tech tree, déblocage de tanks/munitions, sets d’items, corruption late-game, et atlas de campagne persistant.

### Success Metrics (GOTY Ambition)
- **Retention**: ≥55 % des joueurs relancent une run dans les 30 premières minutes.
- **Build Diversity**: aucun top-5 build ne dépasse 45 % des victoires en endgame.
- **Moments Épiques**: ≥1 événement “clipable” (frontline push, convoi sauvé, contre-tech) par run de 12 minutes.

### Stratégie de contrôle
- **P0 absolu** : clavier/souris optimisé (voir `toon/T1_KBM_Gold_Standard.md`). Toute feature de gameplay doit être testée et validée sur ce preset avant d'être envisagée ailleurs.
- **P2 optionnel** : support manette/Steam Deck (voir `toon/T9_Manette_SteamDeck_P2.md`) ne débute qu'après validation du Gold Standard KBM.
- **Règle de non-régression** : aucune tâche ne peut introduire de latence, smoothing ou aim assist implicite côté KBM sans revue dédiée.

---

## 2. Player Experience Breakdown

1. **Combat Feel**: tank piloté clavier/souris prioritaire (KBM Gold Standard), inertie maîtrisée, tourelle indépendante guidée par le curseur et caméra mouse-led. Heat/Fuel/Ammo rétroaction immédiate (SFX, HUD, UI states). La manette reste un add-on P2 optionnel.
2. **Loot Dopamine**: quatre raretés (Commun → Légendaire), affixes tri-ressource et alliés, sets et uniques “twist”, pity timer data-driven.
3. **Procedural Mastery**: chunk library par biome, “vaults” artisanales, chemin critique garanti + poches optionnelles, seed loggable pour QA.
4. **Adaptive Opposition**: contre-technos (boucliers anti-frag, mines EMP, brouilleurs) télémétrées; War Room offre des contre-mesures claires.
5. **Meta Stakes**: atlas persistant, défis rotatifs, contrats partagés et Nemesis Tank (boss ghost d’un autre joueur) en backlog post-MVP.

---

## 3. Production Roadmap

Nous utilisons **TOON** (`/toon/*.md`) pour orchestrer les epics. Chaque epic comporte sprints, tests, checklist QA et dépendances explicites.

### High-Level Timeline (8 sprints · 2 semaines chacun)
1. **Sprint 1 — Boot & Loop**: scène principale, tank de base, ennemi melee, boucle mourir/repartir.
2. **Sprint 2 — Horde & Feel**: cam feel, 4 types d’ennemis, pooling FX, perf target 300×300.
3. **Sprint 3 — Procédural & Objectifs**: générateur de chunks, deux objectifs, boss gate.
4. **Sprint 4 — Loot & Inventaire**: rarités, affixes, pity timer, radial swap.
5. **Sprint 5 — Tri-Ressource & Logistique**: heat/fuel/ammo complets, événements convois.
6. **Sprint 6 — Director & Threat Clock**: métriques build, contre-technos, War Room.
7. **Sprint 7 — Endgame & Meta**: opérations, rifts, World Tiers, sauvegarde tech tree.
8. **Sprint 8 — Polish & Release Candidate**: HUD final, audio/VFX, QA bots, exports CI.

👉 Consulte `toon/README.md` et les fichiers `EPIC_E*.md` pour les user stories, commandes tests obligatoires et artefacts attendus. Les tickets TOON reflètent désormais le focus KBM (T1 — KBM Gold Standard) et un ticket P2 distinct pour la manette/Steam Deck.

---

## 4. Tech Stack & Repo Layout

- **Engine**: Godot 4.x (GDScript par défaut, C# ponctuel si nécessaire).
- **Architecture**: scènes composables, ressources data-driven (`.tres`, JSON/CSV), bus d’événements, object pooling agressif.
- **Tooling CI**: Godot headless tests, export multi-plateformes, batch export Aseprite, télémétrie CSV/JSON.

```
/game
  ├─ project.godot
  ├─ addons/
  ├─ assets/         # art, audio, fonts (voir §6)
  ├─ data/           # items, ennemis, chunks, balance
  ├─ scenes/         # core, player, enemies, world, ui
  ├─ src/            # core, combat, ai, procgen, loot, meta, ui, util
  └─ tests/          # gdUnit, perf harness, bots headless
```

### Coding Conventions
- `PascalCase` pour scènes/nodes, `snake_case` pour scripts GDScript.
- Tick gameplay fixe 60 Hz ; VFX décorrélés.
- Collisions simplifiées (AABB/cercle), pooling pour projectiles/FX.
- Données externes chargées via `ResourceLoader` + cache local.

---

## 5. Contributor & Agent Onboarding

1. **Clone**: `git clone <repo>` puis `cd game` (une fois le Godot project créé).
2. **Ouvrir**: Godot 4.x → *Project Manager* → *Import* → sélectionner `project.godot`.
3. **Lancer**: `F5` (profil Debug). Profilage via *Debugger → Profiler* (viser <16.6 ms p95).
4. **Tests**: `godot --headless --run tests` (configuré en Sprint 1) + scénarios QA (voir epics).
5. **CI locale**: `make verify` (gdformat/gdlint + tests + art-export) dès Sprint 4.
6. **Documentation**: consigner seeds, métriques et captures demandées dans `docs/` (voir TOON).

### Recommended Coding Agents (2025)
| Usage | Agent | Setup Notes |
| --- | --- | --- |
| CLI/CI-first, open-source | **OpenAI Codex CLI** | `npm i -g @openai/codex`, autorisations granularisées (suggest/auto/edit). Idéal pour scripts Godot, Aseprite CLI, workflows GitHub. |
| IDE-native, autonomie VS Code | **Claude Code 2.0** | Extension VS Code (Anthropic). Diffs inline, terminal intégré, MCP/sub-agents pour tâches multi-fichiers. |
| Full IDE agent, gratuit | **Cline (VS Code)** | Open-source, BYO key. Permet plan→edit→run avec prompts “hard tasks”; très utilisé pour pipelines Godot. |
| Premium agent-first IDE | **Cursor Agent Mode** | Composer + Agent Mode pour refactors étendus. Facultatif si l’équipe possède des licences. |

Pour chacun, ajouter les règles du repo (naming, scripts obligatoires) dans leur fichier de configuration (`.codexrc`, `.clauderc`, `.cline/rules.json`, `.cursorrules`).

---

## 6. Asset Production Pipeline (Pixel Art)

**Outil principal**: Aseprite (compilé depuis la source — gratuit). Alternative libre immédiate: Pixelorama.

### Workflow Express
1. Installer Aseprite (build source) ou Pixelorama.
2. Travailler depuis `/assets/templates/` (`tank_base.aseprite`, `tiles_32.aseprite`, `icon_loot.aseprite`).
3. Respecter la palette DB32 (exceptions FX autorisées) et le padding ≥1 px.
4. Exporter via `make art-export` (Aseprite CLI → spritesheet PNG + JSON tags/slices).
5. Import Godot: Filter = Nearest, Mipmaps = Off, Pixel Snap = On, Stretch Mode = `canvas_items`.
6. Vérification QA Art: outline 1 px, slices `muzzle/pivot`, fps d’anim correct, lisible à 70 % zoom.

### Répertoires
```
/assets
  ├─ sprites/      # PNG + .json générés
  ├─ tilesets/
  ├─ ui/
  ├─ fx/
  ├─ palettes/
  └─ templates/
```

### CI Art
- Job GitHub Actions `art-export` (Sprint 4) : cache binaire `aseprite`, export sur fichiers modifiés, publication artefacts + `art_manifest.json`.
- Points durs (muzzle/pivot) récupérés depuis les slices nommées pour aligner les scripts armes.

---

## 7. Systems Reference Sheets

- **Tri-Ressource**: Heat (cadence/surchauffe), Fuel (mobilité/boosts), Ammo (munitions AP/HE/EMP). Événements convois = refills + loot garanti.
- **Loot Tables**: rareté C/R/E/L, iLvl dépendant de World Tier et stage tier, affixes tri-ressource + alliés + sets, corruption late-game (boost + malus).
- **Director**: suit 4 métriques (AOE, portée, burst, mobilité), déclenche contre-technos graduelles, Threat Clock (0→100) menant à contre-offensives.
- **Modes**: Operations (keys + mod cards terrain/ennemis/objectifs), Rifts (timer + boss final), Warfront (atlas persistant hebdo).

Pour le détail complet (JSON exemples, tables, commandes tests), se référer aux epics correspondants (`/toon/EPIC_E*_*.md`).

---

## 8. Instrumentation & QA Gates

- **Telemetry minimale**: durée run, DPS, survie, fps, Threat Clock timeline (CSV). Scripts fournis Sprint 6.
- **Perf targets**: 500 projectiles + 300 ennemis @60 fps (E1), 600 ennemis cumulés @55 fps min (E3), p95 frame <16.6 ms (RC).
- **Test harness**: bots headless (`godot --headless --run qa_bot`) sur 10 minutes, logs archivés.
- **Crash reporting**: envoi local (fichier JSON) + hook CI.

Release Candidate = tous les critères d’acceptation des epics E1→E8 satisfaits + QA art/audio validée.

---

## 9. Resources & Next Steps

- **TOON Roadmap**: `toon/README.md` (chemin critique + commandes à exécuter par sprint).
- **Design Specs additionnels**: compléter `docs/` (procgen, director, endgame) à chaque jalon.
- **Backlog Parking Lot**: features GOTY supplémentaires (Nemesis Tank, Ballistic Beats audio dynamique, sets modulaires avancés) conservées pour post-RC.
- **Community & Inspiration**: consulter Godot 4 forums (destruction persistante, ECS), Discord bullet-heaven devs, Aseprite docs (CLI automation).

**Prochaines actions**
1. Initialiser le projet Godot dans `/game` (Sprint 1, Epic E1).
2. Mettre en place CI minimale (lint + headless) et pipeline art-export.
3. Suivre le chemin critique E1→E8, valider chaque sprint via TOON, documenter seeds/télémétrie.

> Gardons ce README comme la source unique de vérité : toute modification d’objectif, de pipeline ou d’outillage doit y être reflétée immédiatement.
