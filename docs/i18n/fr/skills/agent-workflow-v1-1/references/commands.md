<a name="command-contract-a02"></a>

# Spécification des commandes (A02)

L’assistant, écrit avec la bibliothèque standard, valide les commandes qui passent par `Workflow.execute` et `Workflow.apply`. Il n’intercepte pas les outils natifs, n’authentifie pas leurs reçus ni les messages de l’utilisateur, n’arrête aucun processus et n’observe pas de lui-même les modèles effectivement utilisés. L’agent principal effectue ces opérations et fournit les preuves. Les contrôles A26 d’autorisation, de point de reprise, d’attribution et de capacité restent en vigueur.

Le champ d’état `schema_version: 2` est un entier. Le programme rejette les états sans version, anciens, futurs ou mal formés avant toute modification. Aucun outil de migration ou d’importation d’ancien état n’est disponible. Ouvrez une nouvelle exécution AGWF avec l’autorisation typée actuelle. La création et la restauration d’un point de reprise restent possibles. Une preuve historique ne donne jamais l’autorisation de travailler. Les tests qui injectent une panne utilisent uniquement des copies de test jetables.

<a name="json-and-error-policy"></a>

## Règles JSON et traitement des erreurs

Les fichiers d’état, les fichiers `--input`, l’entrée standard, les métadonnées de point de reprise et le JSON de propriété de l’installateur passent par le même analyseur strict. Les clés en double sont refusées à toute profondeur, même si les valeurs concordent. `NaN`, `Infinity`, `-Infinity` et les littéraux qui semblent finis mais débordent vers l’infini, comme `1e999`, sont refusés. Les fichiers d’entrée acceptent UTF-8 avec ou sans BOM. Pour une commande, la valeur racine du JSON doit être un objet. Une entrée standard vide fournit `{}`. Elle réussit uniquement si la commande l’autorise. `write_json` valide et sérialise avec `allow_nan=False` avant de créer un répertoire ou un fichier temporaire.

Les données d’entrée sont des objets JSON à clés textuelles. Les listes restent des listes. Les nombres entiers et flottants ne servent pas de booléens. Tout indicateur présent doit avoir exactement le type Python `bool`, représenté par `true` ou `false` en JSON. Les chaînes, 0 et 1, null, les listes et les objets ne peuvent pas le remplacer. Omettre un indicateur obligatoire entraîne un refus. Les indicateurs facultatifs prennent false par défaut uniquement lorsqu’ils sont absents. La valeur null est invalide. Les nombres doivent être finis. Les compteurs exigent des entiers supérieurs ou égaux à 1, les horodatages acceptent des entiers ou des flottants entre 0 et sys.float_info.max, et les pourcentages vont de 0 à 100 inclus. La borne supérieure des horodatages exclut les très grands entiers qui feraient déborder les calculs d’horloge. L’horodatage doit aussi respecter les limites de fraîcheur propres à la commande. Il n’y a pas de nouveau plafond de capacité. Le calcul des places inclut toujours l’agent principal et conserve la limite d’agents enfants.

L’assistant vérifie les données avant de créer le verrou ou le répertoire et avant d’exécuter la commande. `apply` travaille sur une copie privée du JSON et la publie uniquement si ses contrôles passent. Les commandes refusées conservent les octets et le SHA-256 de l’état existant, sauf pour le reçu durable INPUT_INVALIDATED et les indicateurs contract_invalid décrits plus bas. Une première commande refusée ne crée aucun état. Un instantané en échec peut, comme auparavant, laisser un répertoire de point de reprise appartenant à l’exécution et non validé. Le cas d’un format valide mais d’une mesure indisponible est une transition volontaire, décrite plus bas.

En cas d’erreur, l’assistant renvoie le code de sortie 1 et émet l’objet suivant sur la sortie d’erreur.

```json
{"error":"exact boolean required","code":"FIELD_TYPE","field":"$.accepted"}
```

| Code | Signification | Champ |
| --- | --- | --- |
| `COMMAND_UNKNOWN` | Commande du moteur inconnue | `$.command` |
| `FIELD_REQUIRED` | Champ obligatoire ou option obligatoire de l’utilitaire manquants | Champ manquant exact |
| `FIELD_UNKNOWN` | Champ inconnu ou option CLI non applicable | Champ incriminé exact |
| `FIELD_TYPE` | Type incorrect fourni par l’appelant | Champ ou indice de liste exact |
| `FIELD_VALUE` | Identifiant invalide, texte ou liste obligatoire vide, valeur numérique hors limites ou état d’inventaire invalide | Champ ou indice de liste exact |
| `JSON_DUPLICATE_KEY` | Membre JSON en double | Membre dupliqué exact, y compris dans un objet imbriqué |
| `JSON_NONFINITE` | Nombre ou jeton non fini, y compris dans les métadonnées brutes | Champ ou indice de liste exact |
| `JSON_SYNTAX` | Syntaxe ou encodage JSON invalide | `$` |
| `JSON_TYPE` | Valeur non représentable en JSON ou clé d’objet autre qu’une chaîne | Valeur ou objet exact |
| `COMMAND_REFERENCE` | Tâche référencée ou champ enregistré obligatoire introuvable | `$.id` pour la recherche. Le message nomme la clé manquante |
| `COMMAND_CONSTRAINT` | Contrôle existant sur l’autorisation, l’état, l’attribution, la revue ou la fraîcheur | Champ concerné si identifiable, sinon `$.command` pour une règle portant sur tout l’état |
| `STATE_VERSION` ou `STATE_TYPE` | Version d’état non prise en charge ou racine d’état autre qu’un objet | `$.schema_version` ou `$` |
| `IO_ERROR` ou `INVALID_VALUE` | Erreur de système de fichiers ou autre erreur de valeur | `$` |

Les chemins utilisent un point pour les clés simples, des crochets pour les autres clés et `[index]` pour les listes. Les erreurs ordinaires d’argparse, comme une commande ou option CLI inconnue ou un argument manquant, conservent sa sortie d’erreur et son code 2. Une option reconnue mais inapplicable à la commande produit l’erreur structurée `FIELD_UNKNOWN` ci-dessus. Aucun des deux modes n’utilise `assert`. La validation fonctionne donc aussi avec `python -O`.

<a name="payload-types"></a>

## Types des données d’entrée

`text` désigne une chaîne encore non vide après suppression des espaces aux extrémités pour la validation. Le contenu d’origine est conservé. `string` peut être vide. `task_id` répond à `[a-z][a-z0-9_]{0,63}`. `strings` est une liste de chaînes non vides, qui peut ne contenir aucun élément. `criteria` exige au moins un élément. `task_ids` est une liste d’identifiants de tâche. `timestamp`, `count` et `percent` suivent les règles numériques précédentes. Seul le suffixe `?` autorise une valeur null explicite.

Chaque commande et chaque objet structuré imbriqué décrit ci-dessous refuse les champs hors de sa liste de champs requis ou facultatifs. Deux exceptions conservent les résultats bruts du fournisseur. Le contenu des relevés et les métadonnées hors mesure dans `rateLimits` et les fenêtres brutes sont traités comme du JSON opaque. Les clés en double et les valeurs non finies ou non JSON restent refusées. Les métadonnées inconnues ne valent ni indicateur de commande ni preuve d’une mesure valide.

| Type imbriqué | Champs obligatoires | Champs facultatifs et contraintes |
| --- | --- | --- |
| `authorization` | `event_id`, `exact_quote`, `kind`, `source_kind`, `source_id`, `goal_id`, `thread_id`, `run_id` de type text. `scope` de type scope. `observed_at`, `source_created_at` de type timestamp. `original_turn_id`, `observed_turn_id` de type text | Aucun autre champ. A26 contrôle les types d’événements autorisés, la provenance directe de l’utilisateur, l’égalité du périmètre, le rejeu et l’âge. |
| `scope` | `kind`, `id`, `objective` de type text | `kind` vaut objective ou report selon les contrôles A26. |
| `source` | `source_kind`, `source_id`, `exact_quote`, `original_turn_id` de type text. `source_created_at` de type timestamp | Doit correspondre aux champs de provenance de l’autorisation fournie. |
| `native_binding` | `binding_id`, `goal_id`, `thread_id`, `run_id`, `objective` de type text. `scope` de type scope. `goal_created_at`, `observed_at` de type timestamp | Rattachement initial N02 uniquement. Le périmètre doit être de type objective et son objectif doit correspondre exactement. La création native doit précéder l’observation. Aucun champ de source humaine. |
| `app_goal` | `goal_id`, `thread_id`, `status`, `source_kind`, `source_id` de type text. `observed_at` de type timestamp | `objective` (text) et `goal_created_at` (timestamp) sont facultatifs sauf si l’autorisation repose sur un rattachement natif. Dans ce cas ils doivent correspondre au rattachement, y compris lors de chaque actualisation. A26 exige une provenance get_goal, une identité concordante, une observation de 60 secondes au plus et l’état actif approprié. |
| `inventory` | Liste d’objets avec `id` et `status` de type text | Aucun autre champ. Identifiants uniques et états running, interrupting, unknown, idle, stopped ou done. Observation datant de 0 à 60 secondes. |
| `models` | Objet associant des noms de modèles non vides à des listes d’efforts de type `strings` | Objet ou liste vide permis. prepare exige toujours l’observation du modèle et de l’effort exacts. |
| `arguments` | Aucun champ obligatoire à ce niveau | Uniquement `task_name` (task_id), `fork_turns` (text), `message` (string), `model` et `reasoning_effort` (text). Record exige en plus l’égalité complète avec les arguments préparés, ce qui empêche l’envoi s’il manque un champ. |
| `receipt` | Objet, liste ou chaîne non vide | JSON opaque issu du résultat réel de l’outil. Aucune affirmation d’authenticité. |
| `rate_limits?` | Objet ou null | primary et secondary facultatifs de type window?. Les autres métadonnées brutes du fournisseur restent opaques. |
| `window?` | Objet ou null | `usedPercent` (percent?), `windowDurationMins` (count?) et `resetsAt` (timestamp?) facultatifs. Les autres métadonnées restent opaques. |

<a name="engine-commands"></a>

## Commandes du moteur

Les listes de champs obligatoires et facultatifs sont exhaustives. Une commande sans champ facultatif n’accepte que les champs requis. Une commande sans entrée reçoit `{}`.

| Commande | Champs obligatoires | Champs facultatifs et valeurs par défaut |
| --- | --- | --- |
| `activate` | `objective` (text), `authorization` (authorization), `interpretation` (string), `source_message` (source), `app_goal_observation` (app_goal), `models` (models), `slot_limit` (count), `inventory` (inventory), `observed_at` (timestamp) | `session_limit` (count?). Absent ou null, aucun plafond supplémentaire de sessions ouvertes. |
| `bind-active-goal` | `objective` (text), `native_binding` (native_binding), `interpretation` et `procedural_check` (text), `app_goal_observation` (app_goal), `models` (models), `slot_limit` (count), `inventory` (inventory), `observed_at` (timestamp) | `session_limit` (count?). État schema2 vierge et inactif, sans tâche, autre champ ni historique. |
| `observe-authorization` | `authorization` (authorization), `interpretation` (string), `source_message` (source) | `app_goal_observation` (app_goal) peut être fourni. Cette commande n’actualise pas l’observation de l’objectif conservée. |
| `app-goal` | `app_goal_observation` (app_goal) | Aucun |
| `usage` | Aucun | `observed_at` (timestamp) et `rateLimits` (rate_limits?). Une omission indique une mesure indisponible. |
| `usage-due` | Aucun | `substantial_work` (bool, false) |
| `inventory` | `inventory` (inventory) et `observed_at` (timestamp) | Aucun |
| `task` | `id` (task_id), `role` (text), `objective` (text), `criteria` (criteria) | `file_inputs` (file_inputs, []), `context_refs` (strings, []), `review_targets` (review_targets, []), `revision` (text, UUID généré si absent), `writes` (strings, []), `depends_on` (task_ids, []), `independent_review` (bool, false) |
| `prepare` | `id` (task_id) | `effort` (text, effort du rôle si absent), `current_context` (context_pin pour N06) |
| `record` | `id` (task_id), `submitted_arguments` (arguments), `session_id` (text), `receipt` (receipt) | `effective_model`, `effective_effort` (text?). Absent ou null signifie non vérifié. |
| `dispatch_failed` | `id` (task_id) et `evidence` (text) | `certain_not_started` (bool, false). Absent ou false laisse l’issue incertaine. |
| `reconcile` | `id` (task_id), `evidence` (text), `no_owned_work_running` (bool) | Exige true et un inventaire récent sans agent enfant actif. |
| `finish` | `id` (task_id), `accepted` (bool), `evidence` (text) | `review_task` (task_id), obligatoire pour accepter une tâche qui exige une revue indépendante. |
| `cancel` | `id` (task_id), `reason` (text) | Uniquement pour une tâche prête qui n’a jamais été envoyée. |
| `stop` | Aucun | Aucun |
| `snapshot` | `writers_quiescent` (bool), `next_action` (text) | `files` (strings, []). Le booléen obligatoire doit valoir true. |
| `restore` | `destination` (text) | Nouvelle destination hors du projet. |
| `halt` | `inventory` (inventory), `observed_at` (timestamp), `owned_processes_stopped` (bool), `stop_evidence` (text) | `degraded` (bool, false). S’il vaut true, `reason` et `next_action` (text) deviennent obligatoires. L’indicateur d’arrêt des processus doit valoir true. |
| `resume` | `authorization` (authorization), `interpretation` (string), `source_message` (source), `app_goal_observation` (app_goal) | Aucun. Exige une continuation correspondante après l’arrêt, des mesures et un inventaire récents et le point de reprise actuel vérifié. |
| `compact` | Aucun | Aucun. Exige des tâches terminales acceptées ou rejetées, aucun agent enfant actif et un point de reprise actuel vérifié. |
| `complete` | Aucun | Aucun. Exige les contrôles de compact et une exécution active. |
| `status` | Aucun | Aucun. Exige un état déjà activé. |

Les fichiers attribués doivent toujours être désignés par des chemins exacts relatifs au projet, conformément aux protections existantes contre les alias Windows, les sorties de périmètre et les liens symboliques. Les rôles et efforts restent soumis au catalogue et aux contrôles de disponibilité existants. Les rôles de vérification et de coordination en lecture seule peuvent déclarer une liste d’écriture vide. Une référence inconnue entraîne un refus. Aucune tâche n’est créée implicitement. `accepted: false` rejette le travail d’un agent qui a cessé d’écrire. `accepted: true` l’accepte uniquement après les contrôles de revue applicables.

<a name="raw-usage-lifecycle"></a>

## Cycle de vie des données d’utilisation brutes

N06 définit `context_pin` par exactement `{path, sha256, manifest_path, manifest_sha256}`. Les chemins sont non vides et relatifs au projet. Les valeurs SHA-256 sont en minuscules. Le manifeste doit être un fichier d’entrée immuable déjà attribué à la tâche. `prepare` compare une vue version1 récente à l’état courant et au manifeste figé séparément. Un message préparé avec un résumé engage ce protocole. `dispatch-start` accepte également `current_context` et exige un nouveau résumé généré sur l’état après la préparation avec le manifeste d’origine. Tous les contrôles natifs restent actifs. Voir [le contexte](context.md) pour le générateur de vues, les preuves qui bloquent le travail, le contrôle de l’auteur et leurs limites.

Dans N07, `status` conserve sa table originale des états de tâche et ajoute `task_observations: {contract_version: 1, tasks: ...}`. L’exécution native et l’acceptation des artefacts sont distinctes. Une tâche dont l’exécution est done reste en attente tant que les contrôles de finish ne sont pas passés. L’acceptation indépendante consigne le lien facultatif existant `review` vers la tâche de revue et exige deux identités de session native non vides et distinctes. Les empreintes des sorties de revue attribuées et consignées doivent aussi concorder. Schema2 reste inchangé. Une revue ne peut pas dépendre directement ou indirectement de l’acceptation de l’auteur dont elle a figé le travail. Les acceptations indépendantes anciennes dont les identités de l’auteur et du vérificateur ou les empreintes de revue sont incomplètes s’affichent comme `historical_unrecorded`. Elles ne satisfont pas une nouvelle dépendance d’acceptation lors de prepare, record ou dispatch-start. Le blocage dans le résumé et les contrôles de l’assistant utilisent le même critère d’indépendance. Aucune migration automatique de l’historique n’est faite.

Les types ou plages mal formés provoquent une erreur et laissent les octets d’état inchangés. Exemples possibles, `usedPercent: "10"`, des compteurs ou horodatages booléens, un pourcentage de 101, une heure de réinitialisation négative, une fenêtre présentée comme une liste ou des métadonnées non finies. Une absence ou un null pour `rateLimits`, une fenêtre ou une valeur de mesure, une mesure bien typée privée de la fenêtre de 300 ou de 10080 minutes, ou une observation par ailleurs valable mais trop ancienne ou future signifient que l’utilisation n’est pas disponible. Cette commande `usage` valide efface volontairement l’ancienne mesure et incrémente `usage_errors`. La première occurrence renvoie `RETRY_USAGE_ONCE`. La seconde renvoie `SAVE_AND_STOP` et passe en cours d’arrêt, sauf si l’exécution est déjà arrêtée ou terminée. Elle ne réactive jamais silencieusement une exécution arrêtée. Les pourcentages 0 et 100 et les pourcentages fractionnaires finis sont valides. 100 déclenche la procédure liée au seuil. Une mesure mal formée n’incrémente pas le compteur des mesures indisponibles. Les champs de métadonnées brutes sont conservés et ne peuvent pas remplacer les fenêtres valides exigées.

<a name="supported-pinned-entry"></a>

## Point d’entrée figé pris en charge

Toutes les opérations passent par `scripts/candidate.py --manifest <manifest> --manifest-sha256 <accepted-pin> --version 1.1`. Les modes d’exécution exigent également `--project <absolute-project>` et `--run-id <fresh-run>`.

| Mode | Entrée | Comportement |
| --- | --- | --- |
| `workflow validate` | Aucune | Vérifie le paquet accepté et les règles des rôles, puis rattache un espace de noms inactif |
| `workflow <typed-command>` | `--input <json-file>` ou entrée standard | Exécute les contrats typés décrits ci-dessus |
| `coordination` | `--input <json-file>` | Exécute un lot borné sans modèle. Aucun envoi natif automatique. |
| `watcher` | `start`, `renew`, `stop`, `status`, `once` | Utilise le moniteur interne figé et l’état appartenant à l’exécution |
| `install` | `--destination .../agent-workflow-v1-1` explicite, `--update` facultatif | Vérifie le contenu du paquet et les conditions de propriété de l’installation |

L’appel direct du CLI de `workflow.py` est refusé. Il ne fournit aucun point d’entrée pour installer, exporter ou générer les rôles natifs. Construisez les fichiers de distribution avec `tooling/build-candidate.py` et installez dans Codex uniquement via l’installateur candidat figé. La conversion d’un schéma historique n’est pas une commande du candidat et ne doit jamais servir à amorcer une installation retirée.

<a name="input-and-state-boundary"></a>

## Limites entre entrées et état

Les rejouements jetables conservent les décisions prises pour des commandes qui donnent les booléens dans le bon format. Les valeurs facultatives peuvent conserver leurs valeurs par défaut sûres à false. Un type incorrect est refusé, même s’il serait interprété comme vrai. L’analyse stricte de l’état protège la syntaxe JSON et exclut les nombres non finis. Les autorisations fournies sous forme de chaîne et les anciens états sont rejetés. Les tests utilisent uniquement des projets jetables.

<a name="typed-assignment-and-source-chronology-a04--gi01"></a>

## Affectation typée et chronologie des sources (A04 / GI01)

Dans N02, `bind-active-goal` rattache un objectif natif actuellement actif après une longue initialisation. Il n’invente pas de source humaine et ne change pas son horodatage. Il exige l’absence d’état ou exactement `{schema_version: 2, status: inactive, tasks: {}}`, les chaînes étant placées entre guillemets dans le vrai JSON. Un assistant migré, terminé, stoppé, arrêté, déjà utilisé ou autrement non vierge ne peut pas se rattacher. L’observation initiale est une preuve réelle de `get_goal` vieille de 0 à 60 secondes. L’objectif, la conversation, le texte de l’objectif, les heures de création initiale et d’observation doivent correspondre au rattachement. L’objectif natif peut avoir été créé longtemps auparavant, jamais après l’observation. L’objectif de la commande et celui du périmètre doivent être égaux. Un périmètre de rapport ou élargi est refusé. Les preuves d’interprétation et de vérification de procédure, distinctes et non vides, consignent le contrôle par l’agent principal du contexte actuel et des restrictions plus récentes. L’assistant ne peut pas authentifier ce contrôle.

L’état conserve `native_binding` comme base actuelle et sépare `native_binding_events` et `native_binding_ids` du registre des autorisations humaines. Chaque événement contient `binding`, `interpretation`, `procedural_check` et `app_goal_observation` d’origine. L’identifiant de rattachement ne peut pas servir ensuite d’identifiant d’événement humain. Les identifiants d’exécution et de périmètre sont réservés avec l’historique existant. Travailler sous ce rattachement exige des observations récentes de l’objectif dont le texte et la création correspondent. L’autorisation humaine conserve son format de source exact et la limite de 300 secondes depuis la source d’origine. Une reprise humaine remplace la base native actuelle et conserve son historique séparé. Une activation humaine ultérieure conserve aussi cet historique. Les enregistrements et réservations natifs mal formés, manquants, dupliqués ou contradictoires entraînent un refus, y compris après une continuation humaine. L’arrêt, l’instantané et la finalisation de l’arrêt s’appliquent toujours. Une nouvelle observation de l’objectif actif ne relance jamais automatiquement un assistant arrêté.

Le schéma 2 refuse le champ de tâche obsolète `inputs`. Une nouvelle validation peut préserver explicitement les chaînes historiques dans `context_refs`. Le programme ne les examine pas pour y chercher une signification de fichier ou de tâche. `file_inputs` est une liste d’objets composés exactement de `{path, revision}`. L’affectation normalise chaque chemin relatif au projet, refuse les doublons après neutralisation de la casse, les sorties du périmètre, les chemins réservés et les liens. Elle fige `sha256` et `present`, avec null et false pour l’absence. Une entrée immuable ne peut pas aussi être attribuée en écriture à cette tâche. Les fichiers à écrire conservent leur propre empreinte de départ avant l’envoi. Chaque tâche possède une révision explicite et une empreinte d’affectation qui couvre son contrat complet. Remplacer une révision d’entrée ou de tâche ou une empreinte puis recalculer en silence ne permet pas d’accepter un ancien résultat.

`review_targets` est une liste d’objets composés exactement de `{producer_id, revision}`, réservée aux vérificateurs. Chaque tâche de production doit exister à la révision précise et avoir une observation récente de sa session à l’état idle, done ou stopped lors de l’affectation et de la revue. L’affectation fige les empreintes exactes et la présence des fichiers de sortie. Les références de contexte ne confèrent pas d’autorisation de revue. Les arguments préparés comprennent toutes les références figées et typées. Les entrées, contrats et sorties des auteurs sont vérifiés de nouveau à prepare, record, finish du vérificateur et à l’acceptation du résultat de l’auteur. Les dépendances acceptées sont également revérifiées. A05 conserve la machine d’états accepted et running. Un auteur soumis à une revue indépendante peut rester en running tout en étant observé à l’arrêt pour les écritures.

Un contrat périmé déclenche `INPUT_INVALIDATED` et un reçu `invalidation` qui contient task_id, la révision d’affectation, le motif, le chemin exact modifié si applicable, l’empreinte ou la révision attendue et réelle, at et affected_task_ids triés. Les tâches qui consomment le même chemin ne deviennent des points de départ de l’invalidation que si leur propre empreinte, l’existence attendue du fichier ou le contrat d’affectation ne correspond pas à l’observation. Une révision plus récente correctement figée sur les octets actuels ou sur l’absence actuelle reste valable si une ancienne empreinte est invalidée. L’invalidation suit ensuite les vrais liens de dépendance et de revue. Les chaînes de contexte et l’égalité des chemins ne créent pas de lien à elles seules. Les états opérationnels restent inchangés. `contract_invalid: true` et le reçu durable empêchent l’acceptation de l’ancienne sortie ou revue. Restaurer les anciens octets ne réactive ni l’ancienne tâche de production ni une revue acceptée. Répéter les contrôles conserve le reçu et les octets d’état.

Lorsqu’il observe une entrée, une sortie de revue ou une sortie de dépendance, le programme refuse avant le calcul des empreintes les liens symboliques, jonctions, fichiers liés en dur, remplacements par un répertoire et sorties de périmètre. Une sortie dangereuse invalide la tâche de production et les revues et consommateurs concernés avant de renvoyer l’erreur. Ce contrôle a lieu à l’affectation de revue, à prepare, record, finish et à l’acceptation du résultat de l’auteur. Le reçu conserve l’empreinte attendue ou null, le chemin relatif exact et une chaîne `actual` commençant par `unsafe: ` suivie du motif contrôlé. Pour un fichier manquant, la valeur est null. Les autres erreurs d’entrée-sortie, comme EIO, remontent sans être classées à tort comme dérive du contrat. Le programme contrôle le chemin observé. Il ne fournit pas un instantané atomique de tout le système de fichiers face à un remplacement externe concurrent.

Toutes les modifications partielles étrangères à l’invalidation sont écartées. En appel direct, `apply` conserve uniquement les enregistrements d’invalidation dans l’état fourni. execute les écrit atomiquement avant de lever l’erreur. Il n’existe aucune commande de réinitialisation ou de recalcul d’empreinte. Examinez le reçu et attribuez explicitement un nouvel identifiant et une nouvelle révision de tâche avec des références figées actualisées. Les autres refus conservent leur règle de préservation des octets.

GI01 exige source_created_at et original_turn_id dans l’autorisation et sa copie exacte des champs de provenance dans source_message. observed_at et observed_turn_id décrivent séparément l’observation. La source doit précéder l’observation et dater de 300 secondes au plus lors de son acceptation initiale. Pour reprendre, l’heure de création de la source doit être postérieure à l’arrêt, à sa finalisation ou à l’achèvement. Une observation plus tardive ne suffit pas. Le dernier repère de source accepté survit à la reprise et à la nouvelle activation. Les heures de création ne peuvent pas reculer. À heure égale, le tour d’origine doit être le même. Les identifiants d’événements et de sources d’origine ne peuvent pas être rejoués en changeant de type, d’heure d’observation, d’exécution, de conversation ou de reprise. Les événements observés doivent correspondre à l’objectif, à l’exécution et à la conversation actuels. Un arrêt explicite doit aussi correspondre au périmètre de l’objectif. Les observations d’intention future, de politique et de rapport peuvent être consignées sans autoriser une exécution ou une reprise de l’objectif courant. Un seuil d’utilisation entraîne l’arrêt mécanique de l’assistant. Il ne modifie pas l’observation de l’objectif de l’application et ne crée pas une autorisation de mise en pause par l’utilisateur.

Ces champs constituent des preuves fournies par l’appelant. Ils n’authentifient pas le fournisseur. L’assistant n’intercepte pas les outils d’objectif natifs directs et n’appelle jamais lui-même update_goal. Le test de non-régression contrôlé avec un arrêt historique, une reprise plus récente puis un rejeu de contexte dans le même tour établit le refus de l’assistant et l’absence de changement dans l’observation de l’objectif actif. N07 ajoute la vue versionnée de l’exécution et de l’acceptation décrite ci-dessus. La mesure du contexte et du cache avec un nouvel agent enfant natif et l’intégration de la pause A06/GI02 attendent toujours des preuves distinctes.
