<a name="lifecycle-startup-and-recovery-implementation-plan"></a>

# Plan de mise en œuvre du démarrage et de la reprise

> Plan historique du 10 novembre 2026. Les listes décrivent les travaux proposés à l’époque. Les autorisations et les éléments privés cités se rapportent à cette intervention. Le [guide d’exploitation](../../../skills/agent-workflow-v1-1/references/operations.md) décrit le fonctionnement actuel.

<a name="goal-and-constraints"></a>

## Objectif et contraintes

Le plan portait sur le nouvel essai après un premier démarrage en échec, l’actualisation des mesures dans les délais, la reprise vérifiée et la clôture normale. Il conservait le candidat figé, le moteur de flux typé et le moniteur sans modèle. L’outil de contrôle devait distinguer la préparation, l’observation réelle de l’hôte, l’exploitation et la clôture. L’essai chronométré restait un mode à demander explicitement. Voir la [spécification historique](../specs/2026-11-10-agwf-lifecycle-reliability.md).

Le plan d’origine autorisait le commit et le push du code source v1.1 concerné. Il excluait la publication d’une version GitHub et la création d’un tag de version. Il exigeait Python 3.11 ou ultérieur, la bibliothèque standard et unittest, les outils Codex existants pour l’objectif et l’utilisation, et les mécanismes Windows de processus et de partage de fichiers. Il préservait le travail en cours, les anciennes preuves d’exécution et les rattachements aux paquets. Le paquet, l’objectif, la mesure, les citations de l’utilisateur et l’identité du processus devaient être observés. Les données synthétiques étaient réservées aux tests isolés. L’expiration du relevé, le quota réel, la réserve anticipée et l’arrêt utilisateur avaient des conditions d’autorisation propres. L’exploitation normale n’introduisait aucun SAVE artificiel. Le plan préservait le travail des autres sessions et les réglages de modèle de l’utilisateur.

<a name="review-focus"></a>

## Points à examiner

1. Un véritable SAVE lié à l’expiration d’un relevé après un premier démarrage raté doit permettre de réessayer si les conditions sont réunies, même avant la création de l’état de l’assistant.
2. Un arrêt pour quota ou demandé par l’utilisateur, un nouvel objectif ou un processus inconnu doit être refusé par le contrôle de reprise technique.
3. Un démarrage partiel ou concurrent reste d’issue incertaine jusqu’à l’obtention de preuves permettant de trancher.
4. Réécrire un relevé ne rajeunit pas l’observation qu’il contient. Une mesure récente peut coexister avec un bail expiré.
5. Un événement natif tardif ne peut pas satisfaire le chronométrage. Une erreur temporaire de partage sous Windows ne démontre pas la fin du processus.

<a name="files-and-responsibilities"></a>

## Fichiers et responsabilités

| Chemin | Responsabilité prévue |
| --- | --- |
| `skills/agent-workflow-v1-1/scripts/candidate.py` | Tentative de démarrage, décision de réessayer, appartenance du moniteur et résultats du cycle de vie |
| `skills/agent-workflow-v1-1/private/limit-watcher/scripts/monitor.py` | Comportement de SAVE, de l’époque et des relevés, avec des changements limités au contrat |
| `skills/agent-workflow-v1-1/scripts/workflow.py` | Règles existantes pour les états actif, arrêté et terminé, et pour l’arrêt utilisateur |
| `tooling/run-lifecycle-check.py` | Outil de contrôle préparé, alimenté par les preuves natives de l’agent principal |
| `tooling/tests/test_lifecycle_check.py` | Tests de préparation, d’événements tardifs et de classement des résultats |
| `skills/agent-workflow-v1-1/tests/test_candidate_coexistence.py` | Intégration depuis le code source et le paquet installé avec de vrais processus Python appartenant à l’exécution |
| `test_native_usage_source.py`, `test_goal_binding.py`, `test_watcher_engine.py`, `test_monitor.py` et `test_watcher_atomic_write.py` | Suites de tests sur les limites des relevés, les objectifs et l’autorisation, les échéances du moniteur et le comportement sous Windows |
| `SKILL.md`, `references/operations.md`, `references/native-usage.md` et `references/coexistence.md` | Instructions d’actualisation, de nouvel essai, d’installation et d’acceptation distincte sur l’hôte |

<a name="task-1-retry-a-failed-first-activation"></a>

## Tâche 1. Réessayer après une première activation ratée

Les interfaces prévues comprenaient `activation_identity(command: str, payload: dict) -> dict`, `record_activation_attempt(runtime, record: dict) -> None` et `startup_retry_options(runtime, engine, command: str, payload: dict) -> tuple[str, ...]`. Le fichier `activation-attempt.json` propre à l’exécution consignait l’identité de la source, `schema_version`, `command`, `pre_state_sha256`, `phase`, ainsi que les observations du moniteur et le détail de l’échec. `prepared`, `failed` et `committed` sont des phases distinctes. Un enregistrement resté à `prepared` exige un examen de l’état réel avant de poursuivre.

- Démontrer d’abord l’échec avec un relevé périmé, puis réessayer avec un relevé récent sous le même objectif pour `bind-active-goal` et `activate`. Préserver l’empreinte du SAVE initial et l’autorisation de l’utilisateur. Vérifier l’état actif et le bon fonctionnement du processus.
- Conserver l’intention de démarrer après `preflight_activation`. Lier la commande, l’exécution, l’objectif natif, la conversation, le périmètre, le texte de l’objectif et l’autorisation. Changer uniquement l’heure d’observation ne crée pas un nouveau périmètre.
- Utiliser, entre le lanceur et les opérations du moteur, un verrou de cycle de vie détenu par l’exécution et réservé à un seul auteur. Refuser un verrou de propriétaire inconnu. L’interface native et l’état Python relèvent de deux opérations.
- Ouvrir une nouvelle époque technique uniquement pour le même objectif et la même exécution après `usage_receipt_expired` confirmé, avec une nouvelle observation réelle de l’objectif actif et un vrai relevé. Un indicateur fourni par l’appelant ne confère aucune autorisation de réessayer. Consigner le succès après vérification du processus et validation de l’état du moteur.
- Tester les refus pour quota, réserve anticipée, autre diagnostic, nouveau SAVE, arrêt utilisateur, objectif modifié ou inactif, processus actif ou inconnu, journal modifié ou incomplet, paquet incorrect et démarrage concurrent. Tester la reprise après un relevé de nouvel essai invalide. Le retour arrière restaure les contrôles précédents vérifiés uniquement dans le périmètre de l’échec applicable et préserve tout SAVE plus récent.
- Exécuter les tests de coexistence du candidat et de relevés natifs. Examiner l’état actif, l’identité actuelle et la fraîcheur, l’historique conservé et la fin réelle du processus.

<a name="task-2-prepare-before-measuring-and-refresh-early"></a>

## Tâche 2. Préparer avant la mesure et actualiser en avance

L’outil devait exposer `prepare`, `run --input PATH`, `observe --input PATH` et `close`. La préparation vérifie le paquet, l’empreinte, les rôles et les commandes avant le démarrage. `refresh_by(at: float, interval: float) -> float` suggère `at + interval - min(15, interval / 4)`. Pour un intervalle de 60 secondes, cela conseille une actualisation à 45 secondes. L’échéance réelle reste la même.

- Vérifier que la préparation et l’aide n’exigent pas d’entrée réelle, qu’une entrée absente ou périmée ne crée pas d’état actif et que la répétition d’un relevé ne peut pas modifier son heure d’observation.
- Obtenir l’objectif et l’utilisation natifs après la préparation. Copier uniquement les champs approuvés, dont les deux fenêtres réelles et leurs dates de réinitialisation. Exclure les données de compte, d’identité et de crédit.
- Distinguer `sample_at`, `next_check_at`, `check_interval_seconds` et `refresh_by`. La publication réussie d’un relevé ou la présence d’un signal périodique ne démontre pas une surveillance à jour et opérationnelle.
- Exiger `status.at == usage.observedAt`, la concordance de l’identité du processus en cours et de l’époque, `sampleFresh` et `leaseValid`. Vérifier le renouvellement après réception du relevé. Ne pas promettre que les relevés resteront récents pendant un appel impossible à interrompre.
- Désactiver l’injection de minuteur en exploitation normale. Vérifier les instantanés quand demandé, puis constater la fin du flux et du processus à la clôture normale. Vérifier que celle-ci n’a créé aucun verrou SAVE artificiel.
- Exécuter les suites du cycle de vie et des relevés natifs avec des heures différentes pour le relevé et sa publication.

<a name="task-3-diagnose-timing-without-confusing-late-events-with-success"></a>

## Tâche 3. Diagnostiquer le chronométrage en tenant compte des retards

- Exécuter `timed-test` uniquement sur demande explicite, avec un processus préparé et opérationnel, un événement véritable et `--event-input PATH --delay-seconds 10 --tolerance-seconds 1`. L’échéance prévue était `event_at + 10` et l’attente utilise une horloge monotone.
- Un événement observé avec 45 secondes de retard doit renvoyer `INVALID_TIMING`. Les essais historiques à 98,74 et 45,05 secondes restent des échecs.
- Utiliser les preuves d’objectif officielles existantes. En l’absence de source d’événements natifs prise en charge, classer l’observation comme tardive ou incomplète selon les faits. Ne pas utiliser de transport privé ou de magasin d’identifiants pour mesurer l’interface.
- Consigner séparément `native_event_at`, `observed_at`, `worker_healthy_at`, `save_requested_at`, `save_returned_at`, `snapshot_verified_at`, `helper_halted_at` et `native_paused_at`. La fenêtre de réussite de 9 à 11 secondes porte sur la demande de SAVE lorsque le processus était déjà opérationnel. La pause native demande sa propre preuve.
- Utiliser les mêmes objectif et exécution lors des deux passages en conditions réelles. Établir que le flux a été actif, sauvegardé et arrêté, puis vérifier une reprise réelle et admissible. Un premier rattachement depuis un état vierge ne démontre pas une reprise.
- Tester les événements synthétiques précoces, tardifs, rejoués et modifiés, l’absence de preuve de pause, un arrêt explicite et les deux passages successifs. Le chronométrage réel demeure facultatif et distinct de l’acceptation normale.

<a name="task-4-verify-recovery-and-regressions"></a>

## Tâche 4. Vérifier la reprise et les régressions

- Tester le premier démarrage, l’expiration pendant le premier essai et sa répétition avec une mesure récente, l’expiration d’une exécution active suivie d’un arrêt, d’un instantané, de la finalisation de l’arrêt et de la reprise. Tester aussi un nouvel essai invalide puis valide, la reprise native de la pause à l’état actif et la clôture naturelle. Utiliser les points d’entrée applicables du code source et du paquet.
- Avant de reprendre, observer `halted` et vérifier l’instantané. Après la reprise, vérifier l’objectif et l’exécution d’origine, la nouvelle époque, l’empreinte du SAVE conservée, l’autorisation et le bon fonctionnement réel du processus.
- Refuser le quota, la réserve anticipée, un nouveau SAVE lié au quota, l’arrêt utilisateur, un autre objectif, les relevés périmés ou rejoués, les points de reprise corrompus et les agents enfants encore actifs dont le sort reste indéterminé. L’état natif `blocked` conserve ses propres critères.
- Dans les tests Windows, copier seulement les fichiers nécessaires appartenant à l’exécution et vérifier l’identité connue du processus et son état final. Une erreur temporaire de partage ne signifie pas que le processus a quitté. Une erreur d’autorisation persistante doit être signalée dans un délai borné.
- Lancer les deux suites du dépôt après réussite des cas ciblés. Constater le nombre de tests à chaque exécution. Le cycle normal en conditions réelles exige un vrai objectif et un outil préparé. L’expiration et le diagnostic chronométré ont leur propre autorisation et peuvent rester à `not_tested`.

<a name="task-5-document-the-contract-and-verify-distribution"></a>

## Tâche 5. Documenter le contrat et vérifier la distribution

- Documenter le premier nouvel essai, la reprise depuis l’arrêt et les procédures propres aux arrêts pour quota ou demandés par l’utilisateur. Expliquer l’ordre d’actualisation de la mesure et de renouvellement du bail et ce qu’établit un diagnostic d’événement tardif.
- Recenser le paquet et le travail existant. Préserver les modifications et les installations en cours d’utilisation. Ne pas déduire qu’une exécution est active à partir d’une ancienne empreinte ni migrer les exécutions actives.
- Construire deux fois à partir d’une source identique. Comparer les empreintes du manifeste et du ZIP. Vérifier tout contenu modifié dans les tests de construction et d’installation. Vérifier le candidat installé avec une exécution passive jetable, contrôler l’empreinte et l’accès Windows, et indiquer la portée passive de l’essai.
- Consigner séparément les preuves de `startup_retry`, `receipt_refresh`, `halted_resume`, `native_continue`, `natural_complete`, `timed_test`, `worker_exit` et `package_match`. Conserver aussi les résultats négatifs antérieurs.
- Avant la publication du code autorisée à l’époque, vérifier l’autorisation, le retour arrière et la préservation des SAVE. Garder les rapports privés, les relevés et l’état d’exécution hors de Git.

<a name="task-6-align-the-existing-v11-identity-and-presentation"></a>

## Tâche 6. Aligner l’identité et la présentation de la v1.1 existante

- Migrer ensemble le nom affiché, le nom d’appel, le chemin du paquet et le nouvel espace de noms. Préserver les anciennes identités d’exécution et les empreintes acceptées.
- Consigner l’historique réel de la v1.0 et les changements de la v1.1 dans `CHANGELOG.md`. Expliquer dans le README le but, les limites, l’architecture, le déroulement et l’orchestration.
- Décrire l’installation manuelle et par agent depuis une copie propre, avec conservation de l’empreinte, vérification et invocation explicite. La préparation ne démarre pas d’objectif natif et ne change pas les réglages de modèle.
- Pour la publication autorisée à l’époque, aligner les champs About et Topics sur le code source. Préserver les accès au dépôt et la protection des branches. Vérifier les commandes du README et la fidélité des diagrammes au code.
- Publier uniquement les fichiers autorisés, vérifier le commit et l’arborescence distants, la version et la documentation, puis confirmer qu’aucune version GitHub ni aucun tag n’a été créé.

<a name="acceptance-boundary"></a>

## Périmètre d’acceptation

Le plan initial plaçait les tâches centrales 1 à 5 avant les travaux de publication finale. Il exigeait de couvrir les cas de premier nouvel essai et de reprise après un arrêt effectif, les tests de non-régression, les contrôles du paquet et les observations réelles dans le bon périmètre. Le comportement de l’interface native nécessitait des preuves de l’hôte. Le résultat à dix secondes constituait une affirmation distincte, qui exigeait un événement natif réel et un chronométrage observé séparément.
