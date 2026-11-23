<a name="repeatable-lifecycle-check"></a>

# Vérifier le cycle de vie de façon reproductible

`tooling/run-lifecycle-check.py` traite de vraies observations d’objectif, d’utilisation et d’inventaire des agents enfants via le candidat figé. L’agent principal obtient ces observations auprès de l’hôte. Gardez la session et ses entrées privées, hors de Git, et attribuez l’écriture de la session à un seul agent.

<a name="prepare-before-measuring"></a>

## Préparer avant de mesurer

Construisez le candidat et conservez l’empreinte externe acceptée. La commande `prepare` vérifie le paquet et enregistre une nouvelle session sans démarrer le candidat ni le moniteur. Terminez la préparation avant de relever la première mesure d’utilisation réelle.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py prepare --candidate C:/build/agent-workflow-v1-1/scripts/candidate.py --manifest C:/build/agent-workflow-v1-1/manifest.json --manifest-sha256 PRESERVED_SHA256 --version 1.1 --project C:/project --run-id check-1 --session C:/private/check-1.json
```

`PRESERVED_SHA256` provient du reçu de construction accepté. Utilisez un projet existant et un fichier de session neuf. Après un changement de paquet, ne rattachez pas une ancienne session à une autre empreinte.

<a name="host-input-object"></a>

## Objet d’entrée fourni par l’hôte

Les commandes `run`, `observe`, `resume`, `close` et `timed-test` acceptent un objet JSON transmis avec `--input`.

| Champ | Contenu |
| --- | --- |
| `activation` | Au premier `run` seulement. L’entrée complète et typée de `bind-active-goal`, avec `native_binding`, `app_goal_observation` correspondant, l’objectif, l’interprétation, la vérification de procédure, les modèles et efforts observés, la limite de places, l’inventaire réel et l’heure d’observation. |
| `app_goal_observation` | Une observation actuelle de `get_goal` avec l’identité de l’objectif et de la conversation, le texte de l’objectif, sa date de création et son état, ainsi que `source_kind: get_goal`, une référence de source réelle et `observed_at`. Ajoutez le véritable `updated_at` s’il est disponible. |
| `usage` | Les champs `source`, `sourceId`, `observedAt` et `result` des [relevés d’utilisation natifs](../skills/agent-workflow-v1-1/references/native-usage.md). Conservez uniquement les fenêtres obligatoires. |
| `inventory` | Identités et états actuels des agents enfants réellement observés. Utilisez `[]` uniquement en l’absence d’agents enfants. |
| `checkpoint_files` | Chemins relatifs à récupérer, qu’ils aient été modifiés, ajoutés ou supprimés, y compris les fichiers de l’agent principal. Le moteur ajoute aussi les fichiers attribués aux tâches. |
| `writers_quiescent` | `true` uniquement après avoir observé que tous les agents en écriture se sont arrêtés. L’outil ne peut pas produire cette observation. |
| `next_action` | Étape concrète à effectuer après la reprise ou la clôture normale. |
| `weekly_exception` | Facultatif au premier `run`. Citation réelle de l’utilisateur dans `exact_quote` et expiration Unix bornée dans `valid_until`. Aucune exception par défaut. |
| `resume` | Entrée complète facultative pour reprendre via le candidat. Sinon l’outil essaie `usage_recovery: true` uniquement après une expiration de mesure consignée. Les arrêts liés au quota ou à l’utilisateur gardent leurs propres conditions. |

Si l’API native ne fournit pas d’identifiant d’objectif indépendant, utilisez une identité dérivée de façon stable, par exemple `threadId:createdAt`, et indiquez qu’elle est dérivée. Les [règles d’exploitation](../skills/agent-workflow-v1-1/references/operations.md) et la [spécification des commandes](../skills/agent-workflow-v1-1/references/commands.md) décrivent les formats typés.

<a name="run-refresh-and-close"></a>

## Exécuter, actualiser et clôturer

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py run --session C:/private/check-1.json --input C:/private/initial-observation.json
python -X utf8 -B tooling/run-lifecycle-check.py observe --session C:/private/check-1.json --input C:/private/new-observation.json
python -X utf8 -B tooling/run-lifecycle-check.py close --session C:/private/check-1.json --input C:/private/final-observation.json
```

Notez l’heure réelle d’observation dès le retour de l’appel natif. L’outil vérifie `status.at == usage.observedAt`, l’identité du processus appartenant à l’exécution, l’époque, le signal périodique, la mesure et le bail. `observe` renouvelle aussi le bail. `refresh_by` suggère une actualisation anticipée sans modifier la véritable échéance.

À `CHECKPOINT_SOON`, vérifiez un point de reprise du travail terminé. Tenez `checkpoint_files` à jour. Une clôture naturelle avec `close` exige une surveillance récente, un instantané vérifié, la fin du flux et l’observation de l’arrêt du processus. Le fonctionnement normal n’injecte aucun événement SAVE.

Au premier `run` avec `weekly_exception`, l’outil installe la règle temporaire du moniteur et renvoie `NEED_POLICY_OBSERVATION` avant d’activer le flux de travail. Obtenez de nouvelles observations d’objectif et d’utilisation, puis relancez la même session. Conservez l’horodatage d’origine de chaque relevé précédent.

<a name="halt-and-resume-after-expiry"></a>

## Arrêter puis reprendre après expiration

Lorsque le moniteur demande SAVE, `observe` applique l’arrêt, l’instantané et la finalisation de l’arrêt, puis renvoie `NEED_POST_HALT_OBSERVATION`. Obtenez de nouvelles observations d’objectif, d’utilisation et d’inventaire après l’arrêt.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py resume --session C:/private/check-1.json --input C:/private/after-halt-observation.json
```

Le moteur utilise le motif réellement observé par le moniteur. La reprise technique exige le même objectif actif et autorisé, des fichiers et un point de reprise inchangés, des preuves récentes postérieures à l’arrêt, et la vérification que le processus appartenant à l’exécution a bien quitté. Les arrêts pour quota, réserve anticipée ou décision explicite de l’utilisateur ont leurs propres conditions de reprise. L’outil ne met pas l’objectif natif en pause, en blocage ni en état actif.

Si la première activation échoue, la session conserve l’empreinte de l’essai de démarrage du candidat. Réutilisez le même `run` avec de nouvelles preuves uniquement si les conditions de nouvel essai sont remplies. Une session modifiée ne constitue pas une preuve recevable. Les essais terminés ou d’issue inconnue ne sont pas automatiquement relancés.

<a name="run-a-timed-diagnosis-only-when-selected"></a>

## Lancer un diagnostic chronométré sur demande

Préparez un processus opérationnel avant l’événement et utilisez `timed-test` uniquement si ce diagnostic est explicitement demandé. Le JSON de l’événement contient `event_id`, `goal_id`, `thread_id`, `source_id`, `event_at` et `observed_at` issus d’une observation native réelle.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py timed-test --session C:/private/check-1.json --input C:/private/current-observation.json --event-input C:/private/native-event.json --delay-seconds 10 --tolerance-seconds 1
```

Un événement arrivé trop tard produit `INVALID_TIMING`. `SAVE_REQUEST_ON_TIME` indique que la demande de SAVE est intervenue dans le délai. La mise en pause native exige une preuve distincte de l’hôte, donc l’outil laisse `native_pause_verified: false` et `native_paused_at: null`. Un contrôle complet en deux passages réels observe d’abord l’état actif, puis l’arrêt, puis un retour à l’état actif après une reprise réelle et autorisée, avec les mêmes identifiants d’objectif et d’exécution. Un test à horloge synthétique couvre seulement la logique.
