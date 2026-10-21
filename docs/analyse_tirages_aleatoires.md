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
- AntaresWeb (AntaREST, branche par défaut) ;
- Antares Simulator (branche `develop`).

Sauf mention contraire, les chemins `generator/…` et `utils/…` sont relatifs à `src/antares/datamanager/`.

Ce document a fait l'objet d'une relecture indépendante, qui a vérifié chaque référence contre le code.

---

## Réponse courte

**Pas complètement, pas avec le code actuel.**

- Antares Simulator ne génère aucune série pour une étude produite par le datamanager. Il existe une seule exception, décrite au §3.a.
- En revanche, **le tirage du numéro de série utilisé pour chaque année Monte-Carlo** reste en partie fait par Antares au lancement.
- Aucun réglage du JSON ne permet aujourd'hui de couvrir tous les cas tout en gardant la variabilité entre années.
- Les trous principaux sont :
  - les clusters thermiques classiques et DSR ;
  - le nucléaire avec chroniques hors FR ;
  - la contrainte couplante « talon » nucléaire quand son groupe diffère de celui de la modulation ;
  - les zones hydro dont seuls les apports ont plusieurs colonnes ;
  - les niveaux initiaux de réservoir.

---

## 1. Tirages faits par le générateur (Python, en amont)

| Série | Où | Graine |
|---|---|---|
| Capacités des liaisons HVDC, et la part HVDC des liaisons hybrides | `generator/generate_link_matrices.py:35-95` (moteur `antares-timeseries-generation`, `MersenneTwisterRNG`) | SHA-256 de `global_seed` + nom de la liaison (`SeedFactory.for_timeseries`). `global_seed` vaut 0 s'il est absent du JSON. Nombre de séries : variable d'environnement `NUMBER_OF_TS_FOR_LINKS`, 60 par défaut |
| Mélange des chroniques SMR (`smr_mixage`) | `generator/generate_nuclear.py:313-338` (`numpy.random.default_rng`) | SHA-256 de `str(smr_mixage.seed)` modulo 2³² (`SeedFactory.from_string`). Aucun tirage si `unit_count == 1` |
| Position et couleur des zones sans `ui` | `utils/area_ui_utils.py` | Aucune graine, mais sans effet sur la simulation |

Précisions :

- **Liaisons :**
  - Le tirage a lieu dès que la clé `hvdcmw*` est présente, même avec un taux de panne nul. On obtient alors 60 colonnes identiques.
  - Le paramètre `enable_random_ts` n'a aucun effet sur les liaisons.
- **Flow-based :** la prédiction par forêt aléatoire (`utils/random_forest_reader.py`) n'effectue aucun tirage : c'est une simple prédiction.

---

## 2. Tirages faits par antares-craft (en amont)

`study.generate_thermal_timeseries(nb_timeseries_thermal)` est appelé dans `generator/generate_study_process.py:105-111`. L'appel a lieu si :

- `enable_random_ts` est vrai ;
- et l'étude contient des thermiques, des DSR, du P2G ou des thermiques ME.

Détails :

- **Clusters concernés :** tous les clusters thermiques, sauf ceux en `gen_ts = force no generation`. Deux cas mènent à ce mode :
  - les clusters nucléaires qui reçoivent des chroniques (`generator/generate_nuclear.py:306`) ;
  - tout cluster thermique, DSR ou ME dont les `properties` du JSON portent `gen_ts: "force no generation"`.
- **Graine :** `settings.seeds_parameters.seed_tsgen_thermal`, 3005489 par défaut.
- **Nombre de séries :** `settings.general_parameters.nb_timeseries_thermal`, 1 par défaut dans craft.
- **Mode local :** craft fait la génération lui-même.
- **Mode API :** c'est la commande AntaresWeb `generate_thermal_cluster_timeseries` qui la fait, avec la même graine. Le résultat est identique si AntaresWeb utilise la même version d'`antares-timeseries-generation`.

Les séries obtenues sont écrites dans `input/thermal/series`.

**Clusters dont les tirages n'ont aucun effet.** Les clusters ME, P2G et AHC (flow-based) n'ont pas de données d'indisponibilité : craft garde la matrice par défaut, avec un taux de panne nul. Leurs N séries sont donc identiques.

> Remarque : craft parcourt les zones et les clusters par ordre alphabétique avec un seul générateur aléatoire. Ajouter ou retirer un cluster modifie donc les séries de tous les clusters suivants.

---

## 3. Tirages qui restent possibles dans Antares Simulator

### a) Génération de séries (load, solar, wind, hydro, thermique) : désactivée

- La clé `generate` de `generaldata.ini` est vide dans les modèles d'étude 9.x d'antares-study-version. AntaresWeb crée ses études à partir de ces modèles, ce qui couvre le mode API.
- En mode local, craft réécrit `generaldata.ini` à la création sans clé `generate`. La valeur par défaut d'Antares est alors « aucune » (`parameters.cpp:294`, `timeSeriesToGenerate = 0`).
- craft (`GeneralParameters`) ne propose aucun champ pour modifier `generate`, donc le JSON ne peut pas l'activer.

**Seule exception :** un cluster thermique dont les `properties` du JSON contiennent `gen_ts: "force generation"`.

- `ThermalClusterProperties(**properties)` accepte ce champ.
- Antares régénère alors ce cluster au lancement (`doWeGenerateTS`, `regenerate_timeseries.cpp`), avec la graine `seed-tsgen-thermal` et `nbtimeseriesthermal` séries.
- Les séries produites par craft sont remplacées **en mémoire**. Les nouvelles séries ne sont écrites sur disque que si l'archivage ou l'export dans l'input est activé.

### b) Tirage des numéros de séries (`seed-tsnumbers`) : toujours fait, et c'est le vrai problème

Pour chaque année Monte-Carlo, `drawAndStoreTSnumbers` (`timeseries-numbers.cpp:150`) tire un indice de série pour :

- load, solar, wind, hydro ;
- chaque cluster thermique, y compris les clusters désactivés, pour lesquels un nombre est consommé sans être utilisé ;
- chaque cluster renouvelable ;
- les liaisons qui ont plus d'une série ;
- les groupes de contraintes couplantes qui ont plus d'une série ;
- les apports STS ;
- les contraintes additionnelles STS.

Antares utilise un seul générateur pour tous ces tirages : ajouter un élément décale donc les tirages de tous les éléments suivants.

**Quand le tirage est remplacé.** Le scenario builder remplace un tirage si les trois conditions suivantes sont réunies :

- `custom-scenario = true` ;
- `derated = false` ;
- la case correspondante est non nulle (0 signifie « aléatoire »).

Il faut aussi une règle `active-rules-scenario`, mais craft et AntaresWeb la renseignent.

**Corrélation inter-modale.** Elle peut réécrire les indices tirés (`timeseries-numbers.cpp:443-478`). Elle est vide dans le modèle d'étude et craft ne l'expose pas, donc elle est sans effet ici.

**Couverture actuelle.** Voici ce que le datamanager remplit aujourd'hui dans le scenario builder, avec une affectation cyclique 1..N répétée sur `nb_years` :

| Catégorie | Couverte ? | Déclencheur ou remarque |
|---|---|---|
| load | Oui | `scenariobuilder."climatic data"` contient `load` |
| hydro | **Partiellement** | `climatic data` contient `hydro`. Mais le nombre de séries est lu sur le seul fil de l'eau (`generator/generate_scenario_builder.py:194-198`), alors qu'Antares tire l'indice sur le maximum de toutes les séries hydro : fil de l'eau, apports du réservoir, débit minimum, puissances max, courbes guides (`series.cpp:193-205`). Une zone dont le fil de l'eau n'a qu'une colonne reste tirée au hasard. C'est le cas aussi des zones virtuelles STEP `w_hydro_open_*` |
| Clusters renouvelables | Oui | `climatic data` contient `wind_onshore`, `wind_offshore`, `solar_pv` ou `solar_thermo` |
| Liaisons HVDC ou hybrides (60 séries) | Oui | Liaisons listées une par une dans `scenariobuilder.links`. Aucun joker n'existe pour les liaisons |
| Liaisons P2G asservies | Oui, si listées | Leur capacité a autant de colonnes que les séries EnR (`generator/generate_p2g.py:500-512`). Elles doivent être listées dans `scenariobuilder.links` |
| Apports STS | Oui | Listés dans `scenariobuilder."sts inflows"` (joker `*` accepté) |
| Contraintes additionnelles STS | Oui | Listées dans `scenariobuilder."sts constraints"` (joker `*` accepté) |
| Contraintes de modulation nucléaire | Oui, sous conditions | Automatique si les contraintes s'appellent exactement `nuc_modulation_hourly`, `_daily`, `_min_weekly` ou `_max_weekly` et ont plus d'une colonne (`generator/generate_scenario_builder.py:482-514`) |
| **Contrainte « talon » nucléaire (`talon_nuc`)** | **Non**, sauf cas particulier | Son second membre a N colonnes (`nbTsColumns`, par exemple 200), et son groupe vient du JSON (`generator/generate_nuclear.py:226-254`). Elle n'est scénarisée que si son groupe est le même que celui des contraintes de modulation |
| Contraintes flow-based | Oui | Automatique (`generator/generate_flowbased.py:640`) |
| Thermique : nucléaire FR, `y_nuc_modulation`, `z_p2g_asservi` | Oui | `scenariobuilder.thermal` |
| **Thermiques classiques et DSR** | **Non** | Ils ont `nb_timeseries_thermal` séries distinctes, donc Antares tire l'indice au hasard |
| **Nucléaire avec chroniques dans une autre zone que FR** | **Non** | `generate_nuclear_availability` s'applique à toute zone (`generator/generate_study_process.py:344-351`) avec N colonnes lues dans un fichier, mais rien ne les scénarise |
| Thermiques ME, P2G non asservi, AHC flow-based | Non, mais sans effet | Leurs séries sont identiques (voir §2) |
| Niveaux initiaux hydro (`hl`) | **Non** | Voir §3.c |

> ⚠️ **Le mode `custom` n'est imposé que dans un seul cas** (`generator/generate_scenario_builder.py:530-532`) : quand l'étude contient les contraintes de modulation nucléaire décrites ci-dessus.
>
> - Dans tous les autres cas, il dépend de `settings.general_parameters.building_mode` dans le JSON.
> - S'il n'est pas à `custom`, Antares ignore **tout** le scenario builder, y compris le câblage flow-based. Toutes les catégories sont alors tirées au hasard.
> - Inversement, ce passage forcé en `custom` écrase un éventuel `building_mode: "derated"` venu du JSON.

### c) Niveau initial des réservoirs hydro (graine `seed-initial-reservoir-levels`)

- Pour chaque zone, Antares tire ce niveau selon une loi bêta entre les courbes guides min et max, au mois d'initialisation (`random.cpp:237`, `randomReservoirLevel`).
- Ce tirage n'a d'effet que si la gestion du réservoir est activée (`reservoirManagement`). Ce réglage est accessible par le JSON via les propriétés hydro (`generator/generate_hydro.py:55-66`).
- Le tirage a toujours lieu, mais sa valeur est déterministe dans trois cas :
  - min = max ;
  - la moyenne est égale au min ou au max ;
  - la clé `hl` du scenario builder est renseignée en mode custom. La valeur tirée est alors simplement écrasée.
- Le générateur ne renseigne jamais `hl`.

### d) Bruits sur les coûts

Ces bruits ne sont pas des séries temporelles. Ils servent à départager des solutions équivalentes.

- **Bruits tirés avec les graines du JSON** (`seed-unsupplied-energy-costs`, `seed-spilled-energy-costs`, `seed-thermal-costs`, `seed-hydro-costs`) :
  - Antares les tire toujours et on ne peut pas les désactiver.
  - Défaillance, déversement et thermique : amplitude entre 5·10⁻⁴ et 6·10⁻⁴ €/MWh, sauf si un « spread » est défini.
  - Hydro en mode `free modulations` : amplitude entre 5·10⁻⁴ et 1·10⁻³.
  - Hydro en mode `minimize ramping` ou `minimize excursions` : c'est la pénalité de variation qui est modulée, d'environ 10 %, et non un coût.
  - Les graines du JSON (`seeds_parameters`) les rendent reproductibles.
- **Bruits à graine fixe dans l'heuristique hydro :**
  - `h2o_j_ajouter_bruit_au_cout.cpp` et `h2o_m_ajouter_bruit.cpp` ;
  - ces bruits sont déterministes et ne sont pas réglables.

---

## 4. Ce que l'on peut faire aujourd'hui uniquement par le JSON

1. Mettre `settings.general_parameters.building_mode: "custom"`.
2. Lister toutes les catégories dans `settings.scenariobuilder` :
   - `climatic data` : `load`, `hydro`, `wind_onshore`, `wind_offshore`, `solar_pv`, `solar_thermo` ;
   - `links` : chaque liaison HVDC ou hybride, et chaque liaison P2G asservie ;
   - `sts inflows` et `sts constraints` : `*`, ou les clusters STS qui ont plus d'une série.
3. Donner à la contrainte « talon » nucléaire le même groupe que les contraintes de modulation.
4. Ne jamais mettre `gen_ts: "force generation"` sur un cluster thermique.

**Ce qui reste hors de portée :**

- **Les clusters thermiques classiques et DSR.**
  - Le seul contournement est `nb_timeseries_thermal: 1`.
  - Il reste alors un tirage d'indisponibilités, mais il est identique pour toutes les années : on perd la variabilité entre années.
- **Le nucléaire avec chroniques hors FR et hors `y_nuc_modulation`.** Aucun contournement n'existe, car ses séries viennent d'un fichier.
- **L'hydro d'une zone dont le fil de l'eau n'a qu'une colonne** alors que les autres séries hydro en ont plusieurs.
- **Les niveaux initiaux de réservoir.** Seul un réglage des courbes guides, par exemple min = max au mois d'initialisation, les rend déterministes.

Le mode `building_mode: "derated"` supprime bien tous les tirages d'indices (`timeseries-numbers.cpp:427-430`). Ce n'est pas une solution, car toutes les séries sont alors remplacées par leur moyenne.

---

## 5. Modifications qui suffiraient à tout faire en amont

1. **Scénariser automatiquement tout ce qui a plus d'une série**, sans dépendre des listes du JSON :
   - tous les clusters thermiques, par exemple avec une option ou un joker `*` dans `scenariobuilder.thermal` ;
   - tous les groupes de contraintes couplantes, contrainte talon comprise ;
   - toutes les liaisons, P2G asservies comprises.
2. **Corriger le comptage hydro.** Calculer le nombre de séries hydro sur le maximum de toutes les séries hydro, et pas sur le seul fil de l'eau.
3. **Imposer le mode custom.**
   - Passer en `BuildingMode.CUSTOM` dès que le scenario builder est rempli, flow-based compris.
   - Refuser `derated` s'il vient du JSON, ou le signaler.
4. **Fixer les niveaux initiaux hydro.**
   - Remplir `hl` pour les zones dont la gestion du réservoir est active.
   - La valeur est une fraction entre 0 et 1, bornée par Antares (`scBuilderUtils.cpp:22-34`).
   - Les courbes guides sont rattachées aux indices de séries hydro (`series.cpp:68`). La valeur doit donc être calculée année par année, à partir de la colonne correspondant à l'indice hydro de cette année. Ce point reste à confirmer.
5. **Verrouiller la génération thermique.** Deux options :
   - après la génération par craft, passer tous les clusters thermiques en `FORCE_NO_GENERATION`, pour qu'Antares ne puisse jamais les régénérer ;
   - ou, plus simplement, refuser `gen_ts: "force generation"` à la lecture du JSON.

Il ne resterait alors que les bruits sur les coûts (§3.d), qui sont minimes et reproductibles.
