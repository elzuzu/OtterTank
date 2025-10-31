## OTTERTANK
# Vision & Choix du moteur

**Moteur : Godot 4.x (open-source, gratuit)**

* **Pourquoi** : 2D performante, pipeline simple, GDScript proche de Python (très “agent-friendly”), scènes/ressources data-driven, exports multi-plateformes sans coûts.
* **Langage** : GDScript (pour rapidité & lisibilité) + C# optionnel si besoin de perf ciblées.
* **Principe d’archi** : composition (scènes + nodes) + ressources (JSON/TRES) + signaux.

**Onboarding “agent de coding” (rapide)**

1. Cloner : `git clone <repo>`
2. Ouvrir avec Godot 4.x → Project Manager → Import.
3. Lancer : `F5` (profil Debug avec overlay perf).
4. Script tooling (optionnel, CLI): `godot --headless --run tests` (voir section CI).

**Arborescence projet (proposée)**

```
/game
  ├─ project.godot
  ├─ addons/            # (eventuellement: gdUnit, remap input, exporters)
  ├─ assets/
  │   ├─ fx/  sprites/ sfx/ fonts/
  ├─ data/
  │   ├─ items/        # JSON: bases, affixes, sets
  │   ├─ enemies/      # JSON: stats, patterns, loot tables
  │   ├─ chunks/       # JSON: librairies de tuiles & chunks procéduraux
  │   └─ balance/      # CSV: courbes, paliers, drop rates
  ├─ scenes/
  │   ├─ core/         # Main.tscn, Game.tscn, CameraRig.tscn, HUD.tscn
  │   ├─ player/       # Tank.tscn (châssis, tourelle), Weapons/
  │   ├─ enemies/      # Slasher.tscn, Shooter.tscn, Elite.tscn, Boss.tscn
  │   ├─ world/        # TileMap, Chunk.tscn, Spawner.tscn, Objectives/
  │   └─ ui/           # Inventory.tscn, WarRoom.tscn, ModCards.tscn
  ├─ src/
  │   ├─ core/         # GameLoop.gd, EventBus.gd, ObjectPool.gd
  │   ├─ combat/       # Damage.gd, HeatFuelAmmo.gd, Projectile.gd
  │   ├─ ai/           # HordeDirector.gd, Steering.gd, SpawnTables.gd
  │   ├─ procgen/      # ChunkAssembler.gd, ObjectivesGen.gd
  │   ├─ loot/         # Rarities.gd, AffixRoller.gd, Pity.gd
  │   ├─ meta/         # Save.gd, TechTree.gd, Unlocks.gd, WorldTiers.gd
  │   ├─ ui/           # HUDController.gd, InventoryUI.gd, RadialSwap.gd
  │   └─ util/         # Math.gd, RNG.gd, DebugDraw.gd, Telemetry.gd
  └─ tests/            # gdUnit tests, perf harness, bots
```

**Standards & conventions**

* Nommage : `PascalCase` pour scènes, `snake_case` pour scripts, signaux `on_…`.
* `ObjectPool` pour projectiles/ennemis/FX; collisions simples (AABB/cercle).
* Tick gameplay fixe (60 Hz), VFX peuvent être à frame rate variable.
* Données externes (JSON/CSV) chargées via `ResourceLoader` + cache.

---

# EPICS (objectifs, livrables, critères d’acceptation)

## EPIC E1 — Core & Framework

**Objectif.** Boucle de jeu stable, pooling, événements, métriques, export debug.

* **Livrables** : `GameLoop`, `EventBus`, `ObjectPool`, `DebugOverlay`, profils d’input (twin-stick), `CameraRig` (shake/zoom), `Telemetry` minimal (fps, counts, frame time p95).
* **Acceptation** : 500 projectiles + 300 ennemis stables à 60 fps sur machine moyenne; aucun alloc >5 ms/frame.
* **Risques** : GC spikes → pooling + `preload` + signal faible; debug draw toggle.

## EPIC E2 — Tank & Game Feel (tri-ressource)

**Objectif.** Mouvement châssis, tourelle indépendante, armes de base, Heat/Fuel/Ammo.

* **Livrables** : `Tank.tscn` (châssis+tourelle), Weapons: cannon/mg/rocket, `HeatFuelAmmo`, HUD jauges, recul, cam shake, aim assist léger.
* **Acceptation** : contrôles manette+souris; feedback (SFX/FX) distinct pour surchauffe/basse munition/low fuel; tutorial hint.

## EPIC E3 — Horde & IA légère

**Objectif.** Ennemis multiples + spawns par tables + steering cheap.

* **Livrables** : 6–8 types de base (melee, ranged, kamikaze, buffer, tanker, elite), `Steering.gd`, `SpawnTables`, `Boss` 1.
* **Acceptation** : 600 ennemis cumulés sans dips <55 fps; patterns lisibles; death FX + carcasses (obstacles temporaires).

## EPIC E4 — Procédural & Objectifs

**Objectif.** Assemblage de stages par chunks taggés + 4 objectifs (Tenir/Escorte/Détruire/Hacker) + 1 boss gate.

* **Livrables** : `ChunkAssembler`, librairie 2 biomes × 12 chunks, `ObjectivesGen`, triggers, chemin critique + poches optionnelles.
* **Acceptation** : 100 seeds → 0 génération bloquée; métrique longueur/variation.

## EPIC E5 — Loot & Itemisation

**Objectif.** Raretés C/R/E/L, affixes, pity timer, inventaire + radial swap.

* **Livrables** : `Rarities`, `AffixRoller`, `Pity`, `InventoryUI`, `RadialSwap`, pickups avec magnétisme/auto-collect, tables de drop data-driven.
* **Acceptation** : 1 000 drops simulés → distribution attendue ±2%; UI claire; 10 légendaires “twist”.

## EPIC E6 — Méta & Sauvegarde

**Objectif.** Tech tree, déblocages tanks/munitions, save/load robuste.

* **Livrables** : `TechTree`, `Unlocks`, `Save` (checksum simple), écran meta.
* **Acceptation** : 100 cycles save/load sans corruption; déblocage persistant après crash simulé.

## EPIC E7 — Director Adaptatif & Threat Clock

**Objectif.** Suivi de build (AOE/portée/burst/mobilité), contre-technos, horloge de menace.

* **Livrables** : `HordeDirector`, `ThreatClock`, `CounterTech` (6 types), `WarRoom` UI.
* **Acceptation** : variation mesurable des spawns selon build; avertissements lisibles; difficulté perçue “juste”.

## EPIC E8 — Endgame (Opérations, Rifts, World Tiers)

**Objectif.** Runs de 8–12 min avec modificateurs et clés; paliers de difficulté & iLvl.

* **Livrables** : `OperationMode`, `RiftMode`, `WorldTiers`, mod cards (terrain, ennemis, objectifs), calcul iLvl et loot luck.
* **Acceptation** : 10 seeds par tier → durée médiane 9–11 min; scaling de loot cohérent.

## EPIC E9 — Alliés & Radio-RTS (MVP)

**Objectif.** 3 unités alliées + 4 ordres radio diegétiques.

* **Livrables** : Drone, Sapeurs, Half-track; ordres *Suivre/Tenir/Harceler/Escorte*; IA cheap.
* **Acceptation** : 30 unités alliées sans perf drop majeur; lisibilité forte.

## EPIC E10 — UI/UX & Accessibilité

**Objectif.** HUD tri-ressource, tooltips, options, remap input, daltonisme, gros caractères.

* **Livrables** : HUD final, inventaire, WarRoom, menus; options audio/vidéo; profils couleurs.
* **Acceptation** : test utilisateurs internes (5 pers) → >80% comprennent sans tuto long.

## EPIC E11 — Audio/VFX & Identité

**Objectif.** SFX “dopamine” (loot, impact), layers musicaux, VFX lisibles; *Ballistic Beats* (toggle).

* **Livrables** : banques SFX, mix simple, 1–2 couches dynamiques; VFX hits/ricochets/cadavres; beat hooks optionnels.
* **Acceptation** : mix non-fatigant; signaux audio distincts par événement critique.

## EPIC E12 — Build/CI, QA & Télémétrie

**Objectif.** Exportants automatiques, crash reports, perf harness, bots QA.

* **Livrables** : GitHub Actions (export Windows/Linux/macOS), `--headless` sim test, logs anonymisés (durée run, DPS, survie, fps), menu debug (cheats).
* **Acceptation** : build nightly téléchargeable; scénario 10 min headless sans erreurs; métriques visibles (CSV/JSON).

---

# SPRINTS (8 sprints de ~2 semaines)

> **Rythme** : 2 semaines, revue jouable à chaque fin de sprint. Chiffrage indicatif (S=1, M=3, L=5, XL=8 pts). Les tâches listées sont principales; ajouter tâches de glue selon besoin.

## Sprint 1 — Boot & Boucle de base

**Objectif.** Lancer une run, bouger/tirer, mourir, recommencer.

* Setup projet Godot + repo + Actions CI (S)
* EventBus, ObjectPool v1 (M)
* Tank châssis + tourelle, mouvement twin-stick (M)
* Arme “canon” + projectiles + dégâts (M)
* Ennemi “melee” simple + spawner (M)
* HUD minimal (HP) + écran GameOver/Restart (S)
  **Démo** : 3 min de survie avec 50 ennemis.
  **Acceptation** : 60 fps stable, aucun crash.

## Sprint 2 — Horde & Feel

**Objectif.** Sensations tank + foule crédible.

* Cam shake, recul, friction directionnelle (S)
* 3 ennemis supplémentaires (ranged/kamikaze/elite) (L)
* Pooling FX (impact, mort) (S)
* Tables de spawn par minute (M)
* HUD HP/Heat/Fuel/Ammo (M)
* Perf target : 300 ennemis, 300 projectiles (L)
  **Démo** : vague 6 min, montée en pression.
  **Acceptation** : frame time p95 < 16.6 ms.

## Sprint 3 — Procédural & Objectifs

**Objectif.** Stages modulaires + 2 objectifs.

* Chunk library (2 biomes × 8 chunks) (L)
* ChunkAssembler (route critique + poches) (L)
* Objectifs : Tenir Zone, Détruire N cibles (M)
* Boss 1 (pattern simple) (L)
* Télémetrie seed (longueur, densité) (S)
  **Démo** : Mission 12–15 min avec boss gate.
  **Acceptation** : 50 seeds consécutifs, 0 blocage.

## Sprint 4 — Loot & Inventaire

**Objectif.** Itemisation fun et claire.

* Raretés C/R/E/L + tables drop (M)
* AffixRoller (+%dmg, cadence, +proj, percée, ricochet) (L)
* Pity timer basique (S)
* InventoryUI + RadialSwap + magnétisme loot (L)
* 10 Uniques “twist” (L)
  **Démo** : montée en puissance sensible via loot; inventaire opérationnel.
  **Acceptation** : distrib. statistique conforme sur 1 000 drops.

## Sprint 5 — Tri-ressource & Events logistiques

**Objectif.** Heat/Fuel/Ammo qui comptent.

* Surchauffe (cadence, jam), low fuel (ralenti), ammo types (AP/HE) (L)
* Pickups & refills (M)
* Événement **Convoi** (risqué → refill + loot miraculeux) (L)
* Équilibrage courbes base (CSV) (M)
  **Démo** : décisions tactiques autour des jauges; convoi sauvé = jackpot.
  **Acceptation** : 70% des testeurs ressentent la tension “ressources”.

## Sprint 6 — Director & Threat Clock

**Objectif.** Adaptation dynamique.

* Metrics de build (AOE/portée/burst/mobilité) (M)
* HordeDirector biaisant spawns (L)
* 6 Contre-Technos + avertissements (L)
* WarRoom (entre vagues) : choix de contre-mesures (M)
  **Démo** : deux runs même seed, builds différents → patterns ennemis différents.
  **Acceptation** : variation mesurable; difficulté perçue juste (sondage interne >75% ok).

## Sprint 7 — Endgame & Méta

**Objectif.** Boucle “grind” et persistance.

* Modes Operation (clé+modifiers) & Rift (timer) (L)
* World Tiers I–III + iLvl loot (M)
* Save/Load (meta + run) + Tech Tree initial (L)
* Écran Meta & progression (M)
  **Démo** : 2 opérations différentes + 1 rift; progression vers Tier II.
  **Acceptation** : runs 8–12 min; sauvegardes résilientes.

## Sprint 8 — Polish, UI, Audio/VFX, Release Candidate

**Objectif.** Lisibilité, identité, stabilité.

* HUD final + options accessibilité (L)
* Mix SFX (loot/impact), VFX ricochets/cadavres (L)
* Balance passe 1 (courbes, loot luck) (L)
* QA bots (headless 10 min) + crash reporting (S)
* Exporters CI (Win/Linux/macOS) (M)
  **Démo** : build RC jouable de bout en bout.
  **Acceptation** : 0 crash connu; p95 < 16.6 ms; tutorial hints compréhensibles.

---

# User Stories clefs (exemples)

* *En tant que joueur*, je peux **déplacer mon tank** et viser indépendamment avec la souris/manette.
* *En tant que joueur*, je **ramasse du loot** qui s’aimante et je vois immédiatement l’impact dans mon build.
* *En tant que joueur*, je **choisis** entre opérations modifiées par des **cards** pour plus de récompenses.
* *En tant que designer*, je **crée un item** en ajoutant un JSON dans `/data/items/` sans toucher au code.
* *En tant que système*, je **tiens 600 ennemis** et 300 projectiles actifs sans lag.

---

# Schémas de données (extraits)

```jsonc
// data/items/base_cannon.json
{
  "id": "cannon_mk1",
  "slot": "turret",
  "rarity": "rare",
  "ilvl": 10,
  "affixes": ["+10%_damage", "+1_projectile", "+8%_heat"]
}
```

```jsonc
// data/enemies/slasher.json
{
  "id": "slasher",
  "hp": 40,
  "speed": 105,
  "damage": 6,
  "tags": ["melee"],
  "loot_table": "lt_basic_t1"
}
```

```jsonc
// data/chunks/urban_set.json
{
  "biome": "urban",
  "chunks": [
    {"id":"crossroads","tags":["steel_floor","cover"],"exits":["N","E","S","W"]},
    {"id":"warehouse","tags":["destructible","loot_rich"],"exits":["E","W"]}
  ]
}
```

---

# CI/CD & Outils

* **Lint/format** : gdformat/gdlint; pre-commit.
* **Tests** : gdUnit + scénarios headless (simulation 10 min) → export CSV perf.
* **Builds** : GitHub Actions → export plateformes; artefacts RC par tag.
* **Télémétrie locale** : JSON anonymisé (durée, DPS, morts, fps) + script de graph.

---

# Risques & parades

* **Perf foule** : pooling, collisions simples, densité capée, culling.
* **Lisibilité** : palette limitée, contours, VFX courts, HUD clair.
* **Équilibrage** : télémétrie + bancs de tests seedés + pity timers.
* **Scope creep** : backlog gelé pendant un sprint; features “plus tard” garées dans EPIC parking.

---

# Roadmap Milestones

* **MVP jouable** : fin Sprint 4 (loot & boss inclus).
* **Beta** : fin Sprint 7 (endgame & meta).
* **RC** : fin Sprint 8 (polish + CI complète).

---

# Prochaines étapes suggérées

1. Initialiser repo Godot 4.x avec arborescence ci-dessus.
2. Créer issues GitHub par user story (labels par EPIC & Sprint).
3. Branch protection + CI minimal (export debug, test headless).

---

# Pipeline Assets Pixel Art — Outil & Workflow (intégré à l’onboarding)

## Outil recommandé (gratuit)

**Aseprite (compilé depuis la source)** — *production-grade, timeline robuste, export spritesheet, tags d’animations, CLI pour CI.*

* **Pourquoi** : pipeline pro (slices, tags, onion skinning), **CLI** puissante pour automatiser exports et atlases.
* **Coût** : binaire payant, **mais build gratuit** depuis la source (licence autorise l’usage personnel du binaire compilé).
* **Alternative 100% libre** : **Pixelorama** (open‑source). Très bon pour sprites/tiles & exports; pas de CLI aussi mature qu’Aseprite mais suffisant pour un workflow manuel.

> **Choix par défaut du projet** : Aseprite (build source) + plugin import Godot `.aseprite` si souhaité. Pixelorama est supporté en fallback.

## Onboarding “artist & agent” (5 min)

1. **Installer Aseprite (source)** : docs officielles → build; ajouter `aseprite` au `PATH`.
2. (Option) Installer **Pixelorama** si vous préférez une GUI libre immédiate.
3. Cloner le repo, ouvrir **Godot 4.x** et le projet.
4. Ouvrir `/assets/palettes/` et `/assets/templates/` (modèles .aseprite & .pxo fournis).
5. Lancer `make art-export` (ou `npm run art:export`) pour tester la chaîne d’export.

## Directories & conventions

```
/assets
  ├─ sprites/          # unités, projectiles, VFX (PNGs, .aseprite)
  ├─ tilesets/         # tuiles (atlas + sources)
  ├─ ui/               # icônes, HUD
  ├─ palettes/         # .gpl/.aseprite palettes (Arne16, DB32, perso)
  └─ templates/        # fichiers modèles (tailles, calques, tags)
```

**Nommer** : `tank_light_idle.aseprite`, `tank_light_move.aseprite`, `turret_rail_attack.aseprite`, `vfx_ricochet_a.aseprite`, `tile_urban_32.png`.
**Animations** (tags Aseprite) : `idle`, `move`, `shoot`, `death` (+ vitesses fps).

## Résolution & style guide

* **Tile size** : **32×32 px** (arènes VS-like) ; sprites joueur ~48–64 px de long (tank) pour lisibilité.
* **Palette** : **DawnBringer-32** (DB32) par défaut; exceptions locales autorisées pour FX (additifs clairs).
* **Outline** : contour 1 px sombre sur unités & loot; VFX <= 6 frames rapides.
* **Rotation tourelle** :

  * *Option A (crisp)* : **16 directions pré‑bakes** (`turret_dir_0..15`).
  * *Option B (souple)* : rotation temps réel Godot (Nearest, pixel snap).

## Export → Godot

**Aseprite CLI (spritesheet PNG + `.json`)**

```
aseprite -b ./assets/sprites/tank_light_move.aseprite \
  --sheet ./assets/sprites/tank_light_move.png \
  --data  ./assets/sprites/tank_light_move.json \
  --sheet-type rows --filename-format {tag}_{frame} \
  --list-tags --ignore-empty
```

* **Tilesets** : exporter atlas 32×32, alignement sur grille.
* **Godot 4 (Project Settings)** :

  * `Rendering > Textures > Default Texture Filter: Nearest`
  * `Rendering > 2D > Use Pixel Snap: On`
  * `Display > Window > Stretch > Mode: canvas_items`, `Aspect: keep`.
* **Import par texture (Dock Import)** : `Filter: Nearest`, `Mipmaps: Off`, `Repeat: Disabled`.
* **SpriteFrames** : glisser le PNG + charger les frames à partir du `.json` ou utiliser le plugin import `.aseprite`.

## Atlas & perf

* **Petits projets** : frames individuelles OK.
* **Milieu/fin** : packer par familles (`player_*.png`, `enemies_*.png`, `vfx_*.png`). Un atlas par biome pour les tiles.
* (Option) Utiliser **Free Texture Packer** (gratuit) pour atlases; sinon Godot `TileSet Atlas` natif.

## Workflow quotidien

1. **Créer/éditer** l’asset dans `.aseprite` (ou `.pxo`).
2. **Tags** d’anim + **slices** pour points d’ancrage (canon, chenilles, point de tir).
3. **Exporter** via script : `make art-export` (batch Aseprite → spritesheet + json).
4. **Commit** : `git add assets/...` (+ message clair `art: tank_light move + shoot v2`).
5. **Godot** : vérifier import (Nearest/No mipmap), assigner dans `SpriteFrames`/`AnimatedSprite2D` ou `TileSet`.
6. **Perf pass** : en jeu, profiler → vérifier surdraw et tailles.

## CI: intégration à la chaîne du jeu

* **GitHub Actions** ajoute un job `art-export` :

  * Cache du binaire `aseprite` (self-host runner conseillé).
  * Batch sur tous les `.aseprite` modifiés → PNG+JSON → artefacts.
* `art_manifest.json` généré (liste des animations, tags, durées) pour les outils de balance/preview.

## Godot ↔ Data

* **Points durs** (barrel, centre rotation) via **slices nommées** dans Aseprite → lus au chargement pour placer le `Muzzle` et `Hitbox`.
* **Couleurs de rareté** (loot) dans palette dédiée (ui_palette.gpl) pour cohérence HUD/icônes.

## Fichiers modèles fournis

* `templates/tank_base.aseprite` (calques: châssis/tourelle/ombres, tags: idle/move/shoot/death, slices: `muzzle`, `pivot`)
* `templates/tiles_32.aseprite` (grille 32, repères auto-tiling Godot)
* `templates/icon_loot.aseprite` (cadres C/R/E/L + gloss FX)

## Alternatives & ressources gratuites

* **Pixelorama** : workflows similaires, exports PNG séquence/spritesheet.
* **Krita** : pour concept art; pas idéal pour anim pixel, mais utile.
* **Kenny.nl / Itch CC0 packs** : placeholders rapides (licence CC0), à remplacer par art maison.

## Bonnes pratiques

* Éviter les obliques > 45° sans AA (staircase visible) → privilégier 16 dirs.
* VFX sur **couches additives** (Godot `CanvasItemMaterial`), limiter tailles.
* Garder **1 px de marge** autour des frames (padding) pour éviter bleeding.

## Check QA Art (definition of done)

* Filtre nearest, mipmaps off ✅
* Padding ≥ 1 px ✅
* Points `muzzle/pivot` présents ✅
* FPS d’anim corrects en jeu ✅
* Sprite lisible à 70% zoom ✅
