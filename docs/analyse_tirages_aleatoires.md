# Tirages aléatoires dans une étude générée par le datamanager

Cette analyse recense les séries temporelles qui peuvent être tirées aléatoirement, et à quelle étape :

- par le générateur (antares-datamanager-generator) ;
- par antares-craft ;
- par Antares Simulator pendant son exécution.

Elle évalue ensuite s'il est possible, avec le code actuel, de configurer le datamanager pour que **tous les tirages soient faits avant l'exécution d'Antares Simulator**.

Sources analysées :

- le code du générateur (branche `main`, commit `e5a793e`) ;
- antares-craft 0.17.1 ;
- antares-study-version 1.0.20 (modèles d'étude vides) ;
- Antares Simulator (branche `develop`).

---

## Réponse courte

**Pas complètement, pas avec le code actuel.**

- Antares Simulator ne génère aucune série pour une étude produite par le datamanager. Il existe une seule exception, décrite au §3.a.
- En revanche, **le tirage du numéro de série utilisé pour chaque année Monte-Carlo** reste en partie fait par Antares au lancement.
- Aucun réglage du JSON ne permet aujourd'hui de couvrir tous les cas. Le trou principal concerne les clusters thermiques « ordinaires ».

---

## 1. Tirages faits par le générateur (Python, en amont)

| Série | Où | Graine |
|---|---|---|
| Capacités des liaisons HVDC, et la part HVDC des liaisons hybrides | `generator/generate_link_matrices.py:35-88` (moteur `antares-timeseries-generation`, `MersenneTwisterRNG`) | SHA-256 de `global_seed` + nom de la liaison (`SeedFactory.for_timeseries`). `global_seed` vaut 0 s'il est absent du JSON. Nombre de séries : variable d'environnement `NUMBER_OF_TS_FOR_LINKS`, 60 par défaut |
| Mélange des chroniques SMR (`smr_mixage`) | `generator/generate_nuclear.py:314-338` (`numpy.random.default_rng`) | `smr_mixage.seed` |
| Position et couleur des zones sans `ui` | `utils/area_ui_utils.py` | Aucune graine, mais sans effet sur la simulation |

La prédiction flow-based par forêt aléatoire (`random_forest_reader.py`) n'effectue aucun tirage : c'est une simple prédiction.

---

## 2. Tirages faits par antares-craft (en amont)

`study.generate_thermal_timeseries(nb_timeseries_thermal)` est appelé dans `generator/generate_study_process.py:105-111`. L'appel a lieu si :

- `enable_random_ts` est vrai ;
- et l'étude contient des thermiques, des DSR, du P2G ou des thermiques ME.

Détails :

- **Clusters concernés :** tous les clusters thermiques, sauf ceux en `gen_ts = force no generation`. Seuls les clusters nucléaires qui reçoivent des chroniques passent dans ce mode (`generate_nuclear.py:306`).
- **Graine :** `settings.seeds_parameters.seed_tsgen_thermal`, 3005489 par défaut.
- **Nombre de séries :** `settings.general_parameters.nb_timeseries_thermal`.
- **Mode local :** craft fait la génération lui-même.
- **Mode API :** c'est la commande AntaresWeb `generate_thermal_cluster_timeseries` qui la fait, avec le même algorithme et la même graine.

Les séries obtenues sont écrites dans `input/thermal/series`.

> Remarque : craft parcourt les zones et les clusters par ordre alphabétique avec un seul générateur aléatoire. Ajouter ou retirer un cluster modifie donc les séries de tous les clusters suivants.

---

## 3. Tirages qui restent possibles dans Antares Simulator

### a) Génération de séries (load, solar, wind, hydro, thermique) : désactivée

- La clé `generate` de `generaldata.ini` est vide dans le modèle d'étude 9.x d'antares-study-version, ce qui couvre le mode API.
- En mode local, craft ne l'écrit pas, et la valeur par défaut d'Antares est « aucune » (`parameters.cpp:294`, `timeSeriesToGenerate = 0`).
- craft (`GeneralParameters`) ne propose aucun champ pour modifier `generate`, donc le JSON ne peut pas l'activer.

**Seule exception :** un cluster thermique dont les `properties` du JSON contiennent `gen_ts: "force generation"`.

- `ThermalClusterProperties(**properties)` accepte ce champ.
- Antares régénère alors ce cluster au lancement (`doWeGenerateTS`, `regenerate_timeseries.cpp`), avec la graine `seed-tsgen-thermal`.
- Les séries produites par craft sont alors remplacées.

### b) Tirage des numéros de séries (`seed-tsnumbers`) : toujours fait, et c'est le vrai problème

Pour chaque année Monte-Carlo, `drawAndStoreTSnumbers` (`timeseries-numbers.cpp:150`) tire un indice de série pour :

- load, solar, wind, hydro ;
- chaque cluster thermique ;
- chaque cluster renouvelable ;
- les liaisons qui ont plus d'une série ;
- les groupes de contraintes couplantes qui ont plus d'une série ;
- les apports STS ;
- les contraintes additionnelles STS.

Ce tirage n'est remplacé que si **`custom-scenario = true`** et que la case correspondante du scenario builder est non nulle (0 signifie « aléatoire »).

Ce que le datamanager remplit aujourd'hui dans le scenario builder, avec une affectation cyclique 1..N répétée sur `nb_years` :

| Catégorie | Couverte ? | Déclencheur |
|---|---|---|
| load | Oui | `scenariobuilder."climatic data"` contient `load` |
| hydro | Oui | `scenariobuilder."climatic data"` contient `hydro` |
| Clusters renouvelables | Oui | `scenariobuilder."climatic data"` contient `wind_onshore`, `wind_offshore`, `solar_pv` ou `solar_thermo` |
| Liaisons (HVDC à 60 séries) | Oui | Liaisons listées dans `scenariobuilder.links` |
| Apports STS | Oui | Listés dans `scenariobuilder."sts inflows"` |
| Contraintes additionnelles STS | Oui | Listées dans `scenariobuilder."sts constraints"` |
| Contraintes couplantes de modulation nucléaire | Oui | Automatique |
| Contraintes couplantes flow-based | Oui | Automatique (`generate_flowbased.py:640`) |
| Thermique : nucléaire FR, `y_nuc_modulation`, `z_p2g_asservi` | Oui | `scenariobuilder.thermal` |
| **Autres clusters thermiques** (classiques, DSR, ME, P2G non asservi, cluster AHC flow-based) | **Non** | Ils ont `nb_timeseries_thermal` séries chacun, donc Antares tire l'indice au hasard |
| Niveaux initiaux hydro (`hl`) | **Non** | Voir §3.c |

> ⚠️ **Le mode `custom` n'est imposé que dans un seul cas** (`generate_scenario_builder.py:530`) : quand l'étude contient des contraintes de modulation nucléaire avec plus d'une série.
>
> - Dans tous les autres cas, il dépend de `settings.general_parameters.building_mode` dans le JSON.
> - S'il n'est pas à `custom`, Antares ignore **tout** le scenario builder, y compris le câblage flow-based. Toutes les catégories sont alors tirées au hasard.

### c) Niveau initial des réservoirs hydro (graine `seed-initial-reservoir-levels`)

- Pour chaque zone, Antares tire ce niveau selon une loi bêta entre les courbes guides min et max, au mois d'initialisation (`random.cpp:237`, `randomReservoirLevel`).
- Le tirage disparaît si :
  - min = max ;
  - ou la moyenne est égale au min ou au max ;
  - ou la clé `hl` du scenario builder est renseignée en mode custom.
- Le générateur ne renseigne jamais `hl`.

### d) Bruits sur les coûts

Ces bruits utilisent les graines `seed-unsupplied-energy-costs`, `seed-spilled-energy-costs`, `seed-thermal-costs` et `seed-hydro-costs`.

- Antares les tire toujours et on ne peut pas les désactiver.
- Leur amplitude est d'environ 5·10⁻⁴ €/MWh, sauf si un « spread » est défini.
- Ce ne sont pas des séries temporelles, et les graines du JSON (`seeds_parameters`) les rendent reproductibles.

---

## 4. Ce que l'on peut faire aujourd'hui uniquement par le JSON

1. Mettre `settings.general_parameters.building_mode: "custom"`.
2. Lister toutes les catégories dans `settings.scenariobuilder` :
   - `climatic data` : `load`, `hydro`, `wind_onshore`, `wind_offshore`, `solar_pv`, `solar_thermo` ;
   - `links` : toutes les liaisons HVDC ou hybrides ;
   - `sts inflows` et `sts constraints` : les clusters STS qui ont plus d'une série.
3. Ne jamais mettre `gen_ts: "force generation"` sur un cluster thermique.

**Ce qui reste hors de portée :**

- **Les clusters thermiques ordinaires.** Le seul contournement est `nb_timeseries_thermal: 1`, qui supprime toute la stochasticité des indisponibilités.
- **Les niveaux initiaux de réservoir.** Seul un réglage des courbes guides (min = max au mois d'initialisation) les rend déterministes.

---

## 5. Modifications qui suffiraient à tout faire en amont

1. **Scénariser tous les clusters thermiques.** Généraliser `_generate_area_thermal_clusters_scenario` à tous les clusters qui ont plus d'une série, par exemple avec une option ou un joker `*` dans `scenariobuilder.thermal`.
2. **Imposer le mode custom.** Passer en `BuildingMode.CUSTOM` dès que le scenario builder est rempli, flow-based compris.
3. **Fixer les niveaux initiaux hydro.** Remplir `hl`, par exemple avec la courbe guide moyenne au mois d'initialisation, ou exposer cette option dans le JSON.
4. **Verrouiller la génération thermique.** Après la génération par craft, passer tous les clusters thermiques en `FORCE_NO_GENERATION`, pour qu'Antares ne puisse jamais les régénérer.

Il ne resterait alors que les bruits sur les coûts (§3.d), qui sont minimes et reproductibles.
