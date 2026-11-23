<a name="lifecycle-reliability-specification"></a>

# Spécification de fiabilité du cycle de vie

> Spécification historique du 10 novembre 2026. Les preuves et autorisations ci-dessous concernent cette intervention passée. Le [guide d’exploitation actuel](../../../skills/agent-workflow-v1-1/references/operations.md) décrit le comportement en vigueur.

<a name="intended-result"></a>

## Résultat attendu

Fiabiliser la première activation, l’actualisation des relevés, la reprise et la clôture normale. Préparer le test avant les exécutions visées. Consigner séparément les résultats observés pour le démarrage, la reprise d’un flux effectivement arrêté et le chronométrage.

<a name="evidence-available-at-the-time"></a>

## Preuves disponibles à l’époque

- Le rapport chronométré privé d’origine relevait l’expiration d’une mesure avant l’activation. Les deux essais à dix secondes ont échoué, à 98,74 et 45,05 secondes. Le démarrage d’une nouvelle époque et le premier rattachement lors du deuxième essai ne démontraient pas la reprise d’un assistant précédemment arrêté.
- Un autre rapport privé, consacré au déroulement normal, consignait un démarrage opérationnel, un deuxième relevé réel 43,423 secondes plus tard, un point de reprise vérifié, une clôture normale et la fin du processus. Cette exécution n’a pas testé un arrêt SAVE.
- À l’époque, `scripts/candidate.py` pouvait ouvrir une nouvelle époque technique et restaurer les données de contrôle après l’échec de `resume`. La première invocation de `activate` ou de `bind-active-goal` n’avait pas le même journal de nouvelle tentative. L’implémentation exigeait des preuves de non-régression distinctes.
- Une maintenance antérieure avait corrigé des conflits d’autorisation et de consignes, ainsi que quatre défauts dans les utilitaires de test et les attentes. Ces comportements devaient rester couverts par les tests de non-régression.

<a name="requirements"></a>

## Exigences

1. Réessayer un premier démarrage sous l’autorisation existante uniquement après un échec `usage_receipt_expired` consigné pour le même objectif et la même exécution, avec un nouveau relevé réel, une observation effective de l’objectif actif et l’absence vérifiée de processus. Cette procédure ne devait exiger ni nouvelle citation de l’utilisateur, ni nouvel identifiant d’exécution, ni réécriture des données d’utilisation.
2. Refuser le quota, la réserve anticipée, l’arrêt utilisateur, un objectif inactif, une appartenance de processus incertaine, un paquet ou des preuves modifiés et un autre objectif. Python vérifie la cohérence et le format. Il n’authentifie pas de manière indépendante la personne ou l’outil désigné par l’appelant.
3. Conserver la visibilité des tentatives échouées. Une exécution partiellement initialisée ne doit pas apparaître active. Préserver les nouvelles traces de SAVE réelles.
4. Préparer le paquet et le test avant la première mesure. Conserver l’heure réelle d’observation. Une écriture plus récente du fichier n’actualise pas une mesure ancienne. Garder `ingest` distinct du renouvellement du bail.
5. En fonctionnement normal, laisser les décisions réelles déclencher `CHECKPOINT_SOON` et `SAVE_AND_STOP`. Ne pas injecter d’arrêt.
6. Démontrer l’action native Continue et la reprise d’un flux véritablement actif puis arrêté, dans un contrôle distinct du premier rattachement.
7. Exécuter le diagnostic chronométré uniquement sur demande. Préparer l’outil avant l’événement natif réel. Une observation tardive ou une préparation qui dépasse l’échéance ne peut pas réussir. La tolérance proposée pour demander SAVE était de 10 ± 1 secondes. Consigner le délai réel.
8. L’agent principal continue d’appeler les outils d’objectif natifs sur l’hôte. Ne pas ajouter d’accès non documenté à l’hôte, de transport privé vers Codex ni de lecture d’identifiants pour le processus d’arrière-plan. Mesurer un clic dans l’interface avec précision exigerait une source d’événements natifs réelle.
9. Vérifier séparément le point de reprise, la fin du processus, l’achèvement de l’assistant et celui de l’objectif natif. Ne mettre en pause qu’avec l’autorisation de l’utilisateur et utiliser `blocked` uniquement selon les critères de l’outil d’objectif natif.
10. Conserver Python 3.11 ou ultérieur et la bibliothèque standard. Une erreur de partage Windows ne prouve ni la fin du processus ni la réussite. Préserver les modifications locales et les anciens enregistrements d’exécution. Ne pas réécrire leurs rattachements.

<a name="original-authorization"></a>

## Autorisation d’origine

Ce document provenait d’une tâche de planification. L’utilisateur a ensuite autorisé, pour cette intervention passée, l’implémentation, le commit et le push du code source v1.1, ainsi que la mise à jour de la documentation. La publication d’une version GitHub et la création d’un tag étaient hors périmètre. L’arrêt chronométré en conditions réelles demeurait un diagnostic à choisir séparément. Les anciens rattachements d’exécution ont été conservés.
