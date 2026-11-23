<a name="agwf-monitoring-follow-up-on-28-october-2026"></a>

# Suivi de la surveillance d’AGWF du 28 octobre 2026

> Compte rendu de maintenance historique. Ce document décrit la suite de la [première intervention](agwf-duzeltmeler-20261028.md). Il ne confère aucune autorisation d’activer AGWF aujourd’hui.

| Constat | Changements et preuves consignés à l’époque |
| --- | --- |
| Les six compétences de rôle pouvaient être sélectionnées automatiquement | Chaque rôle a reçu son propre réglage `allow_implicit_invocation: false` et un périmètre limité aux tâches préparées. Les tests du paquet ont contrôlé les sept compétences. |
| Le code source, le dépôt distant et l’installation différaient | Les outils requis ont été intégrés au paquet. Les fichiers installés ont été comparés au manifeste et au ZIP construits à partir d’un commit source propre. |
| Les lectures et écritures entraient en conflit sous Windows | Les composants concernés ont adopté les lecteurs compatibles avec le partage et les changements de remplacement de fichiers. Des tests sur fichiers temporaires ont couvert les verrous, les nouvelles tentatives et la conservation du contenu précédent. |
| Un second serveur Codex était démarré à chaque mesure | Ce code a été retiré. L’agent principal relève l’utilisation avec l’outil existant de l’hôte, puis Python lit le relevé. Les tests de mesure bloquent les appels qui lancent un processus. |
| PATH pouvait sélectionner un autre CLI Codex | Le choix d’un exécutable a été retiré du chemin de mesure. |
| La sortie d’erreur était perdue | Le serveur enfant a été retiré. Les échecs de mesure conservent au maximum 20 entrées avec le code de diagnostic, la classe d’erreur et l’étape. Les échecs inattendus du processus Python conservent sa sortie d’erreur locale. |
| Deux boucles revérifiaient le paquet toutes les secondes | Le signal périodique vérifie uniquement l’identité de l’exécution et de son époque. Le contrôle complet s’effectue au moment des commandes et des mesures. |
| Les tests utilisaient auparavant un serveur simulé | Ils ont été adaptés au format contractuel des relevés d’utilisation. Le compte rendu mentionne aussi le contrôle d’une vraie réponse Codex par l’intermédiaire d’un fichier temporaire. |
| Les règles du rôle indiquaient encore un seuil erroné | L’ancien texte `%10` a été aligné sur le seuil d’arrêt effectif de `%20`. Un démarrage refusé faute de données efface son indicateur d’activité. |

Le nouveau chemin de données transmet l’heure d’observation d’origine et l’identité de la source avec `--usage-input`. Le moniteur rattache le relevé à son exécution et à son époque. Il refuse les relevés absents, futurs, expirés, antérieurs à ceux déjà observés ou qui désignent une autre exécution. Publier une nouvelle fois le même relevé ne peut pas actualiser son horodatage. Si aucune donnée récente n’arrive, le moniteur réessaie de valider les données, puis consigne SAVE_AND_STOP.

L’agent principal obtient de nouvelles observations avant l’échéance. Python n’appelle pas l’outil de l’hôte. `ingest` publie uniquement les données. L’activation, le lancement du processus, l’autorisation, le renouvellement de l’observation de l’objectif et la levée d’un arrêt sont des opérations distinctes.

Le compte rendu signale 111 tests ciblés en mode standard et avec `-O`. Cette suite interdisait les vrais lancements d’AGWF ou de Codex. Le constructeur de paquet et les tests des outils en lecture seule pouvaient utiliser les sous-processus autorisés. Les contrôles du code source, les empreintes des fichiers installés, la désactivation des compétences et la reproduction depuis un code source propre ont été consignés séparément.

Selon ce même compte rendu, un résultat réel d’utilisation de l’hôte est passé par le mécanisme des relevés sans créer de processus supplémentaire ni modifier l’heure d’observation. Aucune donnée d’identité ou de compte n’a été ajoutée au paquet. La cause exacte de l’ancien problème avec Remote est restée indéterminée et la coexistence réelle d’AGWF et de Remote n’a pas été testée. Voir la [spécification des relevés d’utilisation natifs](../skills/agent-workflow-v1-1/references/native-usage.md).
