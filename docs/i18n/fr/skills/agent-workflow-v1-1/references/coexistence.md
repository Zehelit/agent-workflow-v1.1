<a name="pinned-runtime-binding-and-installation"></a>

# Rattachement figé de l’exécution et installation

L’identité technique est `agent-workflow-v1-1`, le nom affiché `agent-workflow-v1.1`. L’invocation implicite est désactivée. Ce candidat s’utilise sur sélection explicite. L’utilisateur garde la maîtrise du modèle du client principal. Les rôles du flux peuvent utiliser uniquement `gpt-6.1-sol`, `gpt-6-astra` ou `gpt-6.1-terra`, avec un effort au moins égal à `medium`. Le rôle privé de surveillance est fixé à Terra/medium. Aucun remplacement automatique n’est permis.

Une sélection explicite actuelle oblige l’agent principal à démarrer et vérifier l’exécution délimitée dans le même tour. Elle peut la réactiver après désactivation. Une ancienne sélection ou une intervention sur le paquet ne suffit pas. Lorsque les preuves d’activation complète manquent ou que la création d’un objectif natif est interdite, démarrez manuellement le moniteur comme indiqué dans le [guide d’exploitation](operations.md). Attendez un démarrage vérifié avant le travail de fond. Ne passez pas discrètement à un travail direct sans surveillance. Toute consigne d’arrêt ultérieure reste prioritaire.

Construisez avec Python 3.11 ou ultérieur et la bibliothèque standard.

```powershell
python -X utf8 -B tooling/build-candidate.py --output C:/staging/build-one
```

La sortie contient `agent-workflow-v1-1/`, un ZIP déterministe et `build-receipt.json`. Conservez la valeur `manifest_sha256` du reçu dans les preuves acceptées de coordination ou de transmission, en dehors du paquet modifiable. L’empreinte fournie par l’appelant sert de référence de confiance. Les contrôles complets du manifeste et des dépendances précèdent le rattachement, les écritures importantes de l’état et les étapes de coordination. Ils s’effectuent aussi à chaque cycle de mesure du processus. Le signal périodique vérifie uniquement les données de contrôle de l’exécution et de son époque. Il ne parcourt pas et ne hache pas le paquet à chaque seconde. Recalculer le SHA d’un manifeste modifié et passer la nouvelle valeur ne fournit pas d’attestation externe. Python vérifie l’intégrité des octets et l’identité. Il ne peut pas authentifier une approbation humaine, les métadonnées du fournisseur ou du modèle, ou un système de fichiers contrôlé par l’appelant. Un exécutable ou interpréteur compromis, ou un appelant qui modifie à la fois le rattachement et l’empreinte externe acceptée, se trouve hors de ces garanties.

Placez les options globales avant le mode. L’exécution depuis le dépôt utilise `skills/agent-workflow-v1-1/scripts/candidate.py` et le manifeste construit. L’exécution installée utilise `scripts/candidate.py` et le manifeste installés, avec la même empreinte externe conservée. Chaque configuration source ou installée utilise un identifiant d’exécution distinct. Le rattachement immuable inclut les chemins résolus des dépendances et du manifeste, le projet, l’exécution, la version et les empreintes. Le candidat refuse le déplacement vers un autre assistant, le remplacement du paquet, les dépendances modifiées ou absentes et la réutilisation d’un identifiant avec une autre empreinte. Démarrez une nouvelle exécution après avoir approuvé une mise à jour.

```powershell
$pin = '<preserved manifest_sha256 from accepted build receipt>'
$entry = 'C:/staging/build-one/agent-workflow-v1-1/scripts/candidate.py'
$manifest = 'C:/staging/build-one/agent-workflow-v1-1/manifest.json'
$common = @('-X','utf8','-B',$entry,'--manifest',$manifest,'--manifest-sha256',$pin,'--version','1.1','--project','C:/project','--run-id','candidate-run-1')
python @common workflow validate
python @common workflow status
python @common coordination --input C:/project/batch.json
```

Une invocation autorisée de `workflow activate` ou de `bind-active-goal` démarre le moniteur et vérifie sa première mesure opérationnelle, transmise par `--usage-input`. `resume` ouvre une nouvelle époque, avec un nouveau relevé natif, après une reprise valide ou une récupération contrôlée d’un relevé expiré sous l’autorisation d’un objectif toujours actif. Le [guide d’exploitation](operations.md) présente les chemins de reprise et les [relevés natifs](native-usage.md) détaillent la mesure. Aucun serveur ni CLI Codex n’est lancé pour mesurer l’utilisation. Les commandes actives renouvellent le bail. `complete` et `halt` arrêtent le processus Python appartenant à l’exécution et vérifient sa fin. Les commandes explicites `watcher` suivent les mêmes règles de relevé.

`workflow` accepte les commandes typées du moteur et `validate`. Les commandes lisent le JSON avec `--input` ou par l’entrée standard. Les anciennes interfaces du moteur `native/install/export/migrate` ne sont pas prises en charge par ce candidat. L’autorisation de `activate` et les preuves de `bind-active-goal` doivent porter l’identifiant d’exécution choisi. `validate` vérifie les rôles et dépendances et rattache l’espace de noms sans activer d’objectif. Un lot a la forme `{ "batch_id": "inspect", "steps": [{ "key": "status", "command": "status", "payload": {} }] }`. Il comprend au plus 64 étapes, un seul auteur principal et un journal durable des résultats en attente. Il ne relance ni n’envoie de tâche native automatiquement. Les commandes de fond exigent une autorisation typée actuelle et les preuves d’acceptation applicables. Un inventaire terminal ne vaut pas acceptation du résultat. Une tâche périmée dont l’agent a cessé d’écrire peut être écartée avec `finish accepted:false`. Un agent encore en écriture doit d’abord s’arrêter. La compaction exige un point de reprise vérifié et archive l’historique des tâches et des invalidations.

La racine isolée est `<project>/.agent-workflow-v1-1/runs/<run-id>/`. Les racines de l’état opérationnel sont réservées et ne peuvent pas être attribuées en écriture aux tâches. Le moniteur intégré utilise `watcher/` sous cette racine, un utilitaire de planification explicite et figé, des données de propriété immuables et les arguments de relance du point d’entrée. Les contrôles d’une autre exécution ou un appel direct de l’interface privée ne peuvent pas renouveler ni arrêter un autre processus. Une conversation inactive ne renouvelle pas le bail à durée limitée. Utilisez une commande active du candidat aux étapes clés. Une mise à jour ne migre pas silencieusement les processus en cours. Il n’existe ni compétence ni lanceur séparé pour le moniteur. Les dépendances privées sont chargées depuis le paquet approuvé.

Une première installation, une vérification sans écriture ou une mise à jour gérée exigent une destination explicite dont le nom se termine par `agent-workflow-v1-1`.

```powershell
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination C:/staging/skills/agent-workflow-v1-1
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination C:/staging/skills/agent-workflow-v1-1 --update
```

Les mises à jour exigent une installation appartenant à l’outil dont les octets n’ont pas changé. L’installateur valide le paquet préparé avant de remplacer le répertoire existant. Si la mise en place échoue, il restaure le dossier précédent. Si la mise à jour réussit, il supprime les données temporaires de la transaction. Il ne fournit ni archive permanente ni commande de retour arrière. Si la restauration échoue à son tour, le dossier de transaction contient la seule copie précédente pour une récupération explicite. Ne le supprimez pas avant d’avoir vérifié la récupération. Les commandes préparent seulement les fichiers, comme l’indique `activated:false`. La visibilité au catalogue, la sélection explicite de la compétence, les métadonnées du modèle effectivement utilisé par le fournisseur et l’acceptation du produit ou de l’essai pilote en conditions réelles restent à vérifier après l’installation.

Sous Windows, l’installateur réinitialise la liste de contrôle d’accès du répertoire temporaire pour qu’elle hérite des droits du dossier parent de la destination avant la copie des fichiers. Il ne désigne aucun compte Windows en particulier. Un autre compte ou un compte renommé reçoit les droits déjà attribués à son répertoire parent `skills`. Ce dossier doit donner à l’utilisateur prévu les droits de lecture et d’exécution. Après l’installation, vérifiez la découverte et l’exécution directe du candidat depuis une session Codex ordinaire, sans élévation. Un succès dans une session élevée ne suffit pas.
