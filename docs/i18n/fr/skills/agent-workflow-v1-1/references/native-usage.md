<a name="native-usage-receipts"></a>

# Relevés d’utilisation natifs

Utilisez cette procédure dans une exécution AGWF activée et explicitement autorisée. Une invocation explicite actuelle impose le démarrage dans le périmètre autorisé. Si la création d’un objectif natif est interdite, utilisez la surveillance manuelle décrite dans le [guide d’exploitation](operations.md). Mettre à jour le paquet, discuter de la compétence ou publier des données d’utilisation ne réactive pas à lui seul un flux désactivé.

L’agent principal appelle `mcp__codex_app__get_usage_limits` à travers la connexion Codex existante. Python n’ouvre pas de second app-server, ne choisit pas un autre CLI, ne lit aucun fichier d’authentification, ne se rattache pas à un transport privé et n’appelle pas de modèle. Le processus d’arrière-plan surveille les relevés reçus et contrôle leur fraîcheur. Aux étapes clés du travail, l’agent principal doit appeler l’outil de l’hôte.

Construisez un fichier JSON d’entrée privé à partir d’un résultat réel de l’outil.

```json
{
  "source": "mcp__codex_app__get_usage_limits",
  "sourceId": "actual-tool-call-receipt-id",
  "observedAt": 1800000000,
  "result": {
    "rateLimits": {
      "limitId": "codex",
      "primary": {"usedPercent": 25, "windowDurationMins": 300, "resetsAt": 1800010000},
      "secondary": {"usedPercent": 30, "windowDurationMins": 10080, "resetsAt": 1800600000}
    }
  }
}
```

Les chiffres ci-dessus sont des exemples. Conservez la véritable heure d’observation, l’identité du relevé et ses valeurs. Ne les inventez pas et ne remplacez pas l’heure d’observation par l’heure de publication du fichier. Lorsque le résultat réel contient plusieurs catégories, utilisez en priorité `rateLimitsByLimitId.codex`. Si la table fournie ne contient pas Codex, l’entrée est refusée, sans se rabattre silencieusement sur une autre catégorie. Les deux durées de fenêtre et leurs dates de réinitialisation sont obligatoires. Copiez dans cette entrée privée uniquement les champs de mesure nécessaires. Les champs de compte, de profil et de crédit sont superflus. Le fichier `watcher/usage-source.json` publié conserve uniquement les champs approuvés des fenêtres, l’identité de l’observation et de sa source, et le rattachement à l’exécution et à son époque.

`workflow activate --input AUTH.json --usage-input USAGE.json` et `bind-active-goal` vérifient l’autorisation avant de démarrer un processus. `resume` exige également un nouveau relevé transmis avec `--usage-input`. Un relevé d’une époque précédente ne peut pas être réutilisé. La surveillance manuelle avec `watcher start --usage-input USAGE.json` suit les mêmes contrôles préalables. Si le relevé manque ou a déjà expiré, le démarrage est refusé avant la création du processus.

Pour publier une mesure récente pendant le travail, utilisez `watcher ingest --usage-input USAGE.json` ou l’option `--usage-input` d’une commande de flux de travail ou de coordination. `ingest` n’active pas de flux de travail, ne démarre pas de processus, ne renouvelle pas l’autorisation et ne lève pas un arrêt SAVE enregistré. Un nouveau fichier source déclenche une évaluation avant l’ancienne échéance. L’état peut continuer à afficher brièvement l’évaluation précédente pendant que le processus lit l’entrée. Attendez qu’un état récent confirme que la surveillance est opérationnelle avant un envoi important.

L’agent principal obtient un nouveau relevé réel avant l’échéance adaptative, de 300 secondes au maximum, et plus courte si le seuil approche ou si l’utilisation augmente rapidement. Relire un ancien fichier ne constitue pas une nouvelle mesure. Si l’agent principal ne parvient pas à obtenir de données récentes, le processus retente une fois la validation, puis verrouille l’état SAVE_AND_STOP. Après un arrêt dont le motif `usage_receipt_expired` a été consigné, `workflow resume` avec `usage_recovery: true` peut rétablir le même objectif toujours actif et déjà autorisé. Voir la [reprise technique](operations.md#expired-telemetry-recovery-under-existing-authority). De nouvelles données ne lèvent pas à elles seules le verrou d’arrêt. Cette reprise contrôlée n’exige pas un nouveau message de l’utilisateur. Les arrêts pour quota réel ou sur demande explicite de l’utilisateur gardent leurs propres conditions de reprise. Actualisez le relevé aux étapes clés en gardant assez de temps pour le prochain appel. L’échéance ne donne pas un délai d’attente recommandé. Les appels longs et impossibles à interrompre limitent le renouvellement via l’outil hôte. Aucun accès autonome en arrière-plan à l’outil natif n’est garanti.

Le programme refuse les entrées mal formées, l’exécution ou l’époque incorrecte, les observations futures ou périmées, les horodatages qui reculent et les relevés modifiés puis rejoués. Republier un relevé identique ne change rien. L’heure et la source proviennent de l’appelant, comme pour les autres relevés natifs. La seule forme du JSON n’authentifie pas le fournisseur. Gardez les preuves brutes privées. Une étiquette de source arbitraire ne vaut pas preuve indépendante.

Les échecs de mesure exposent un `diagnosticCode` issu d’une liste autorisée et une classe d’erreur dans la mesure et l’état. Ils conservent au maximum 20 entrées structurées dans `watcher/diagnostics.json`. Le texte des exceptions, les données d’authentification et le contenu arbitraire des outils sont exclus. Les échecs inattendus du processus Python sont toujours consignés localement dans `worker-stderr.log`. Il n’y a plus de sortie d’erreur de serveur enfant à perdre, puisque ce serveur n’existe plus dans ce mécanisme.

L’intégrité complète du paquet est contrôlée à chaque commande du candidat et à chaque évaluation du processus, y compris lors de la publication d’une nouvelle entrée. Les signaux périodiques vérifient seulement les données de contrôle de l’exécution et de l’époque et publient leur état. Ils ne parcourent pas tout le paquet et ne recalculent pas toutes ses empreintes. Une dépendance modifiée sera refusée à la prochaine évaluation ou commande. La détection n’est pas garantie à chaque signal.

Les tests unitaires couvrent la publication et la lecture de vrais fichiers de relevés, le JSON strict, l’expiration, la réutilisation, l’identité, les seuils, les diagnostics bornés et l’absence de démarrage de processus sur le chemin de mesure. Le fonctionnement réel dans Remote exige des observations distinctes. Gardez le flux installé désactivé jusqu’à ce que l’utilisateur le réactive explicitement. Une intervention de maintenance ne doit pas lancer des processus AGWF réels pour fabriquer cette preuve.
