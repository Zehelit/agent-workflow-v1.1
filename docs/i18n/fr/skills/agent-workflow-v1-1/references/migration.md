<a name="state-compatibility-and-recovery-boundaries"></a>

# Compatibilité des états et limites de la reprise

La version d’exécution 1.1 accepte uniquement l’état courant au format schema2 dans un espace de noms approuvé et immuable. Un état inconnu, endommagé ou d’une ancienne version est refusé sans modifier l’état opérationnel. Les anciens textes d’autorisation, les reçus de tâche archivés et les observations d’objectifs périmées n’accordent aucune autorisation.

Le point d’entrée figé `scripts/candidate.py` ne fournit pas de commande de migration. Le moteur ne contient ni convertisseur d’état ancien ni validateur des anciennes autorisations. L’importation automatique, le démarrage à partir d’une autre installation ou le retour à son paquet ne sont pas pris en charge. Ouvrez une nouvelle exécution après avoir approuvé une mise à jour du paquet. Le manifeste et le rattachement à l’assistant de l’exécution courante doivent rester les mêmes.

La reprise actuelle repose sur le point de reprise vérifié de l’exécution acceptée et sur une autorisation utilisateur nouvelle et valide. Conservez les horodatages des sources, les identifiants réservés contre la réutilisation, l’intégrité des fichiers et l’observation qui confirme l’arrêt des agents en écriture. Les anciens textes d’arrêt ou de reprise restent historiques. L’assistant n’active pas automatiquement l’objectif récupéré.

Les nouveaux points de reprise appartiennent à `agent-workflow-v1-1`. Le répertoire d’état par défaut du moteur seul est `.agent-workflow-v1-1`. Le point d’entrée pris en charge, rattaché à un manifeste figé, utilise `.agent-workflow-v1-1/runs/<run-id>`. Les anciens répertoires `.agent-workflow` et `.agent-workflow-v1-0` ne sont pas importés. Ils restent réservés et ne peuvent servir ni de chemins de tâche ni de destinations de restauration. Le nettoyage actuel des instantanés conserve leurs marqueurs de propriété. Un appel direct de l’interface en ligne de commande du moteur est refusé.

<a name="move-the-user-installation-to-the-11-technical-identity"></a>

## Passer l’installation utilisateur à l’identité technique 1.1

1. Recensez les processus et les agents en écriture. Arrêtez uniquement le travail qui vous appartient. Avant de déplacer une ancienne installation, vérifiez l’arrêt du travail qui vous appartient et la fin effective du processus.
2. Vérifiez l’ancien paquet par rapport à son registre de propriété. Vérifiez les empreintes d’une sauvegarde située en dehors de tous les répertoires où Codex découvre des compétences. Conservez les modifications éventuelles avant de poursuivre.
3. Installez le nouveau paquet dans `skills/agent-workflow-v1-1` avec son nouveau manifeste et l’empreinte conservée à l’extérieur. N’utilisez pas `--update` sur une identité technique plus ancienne.
4. Avant de modifier les chemins de découverte, exécutez `workflow validate` depuis le nouveau point d’entrée installé, avec un nouvel identifiant de vérification unique. Comparez l’identité, la version, l’espace de travail et l’empreinte du manifeste renvoyés. Cette opération crée `run-binding.json`. Elle ne démarre ni l’objectif, ni l’état du flux de travail, ni le moniteur.
5. Le nouveau nom d’appel est `$agent-workflow-v1-1`. Une fois la validation directe réussie, mettez à jour les chemins des compétences principale et de rôle dans Codex en conservant les réglages d’activation et d’invocation explicite. Protégez les modifications de configuration concurrentes. Vérifiez les processus qui vous appartiennent de nouveau et préservez toute installation active utilisée par une autre session. Après la transition, archivez l’ancien paquet en dehors de la découverte des compétences, puis vérifiez la configuration et la sauvegarde.
6. Conservez les identités, chemins et empreintes des anciens points de reprise, rattachements d’exécution et reçus. Démarrez une nouvelle exécution avec le nouveau paquet. Pour reprendre un travail inachevé pertinent, transmettez séparément à cette nouvelle exécution les résultats antérieurs vérifiés et l’autorisation actuelle réelle.
