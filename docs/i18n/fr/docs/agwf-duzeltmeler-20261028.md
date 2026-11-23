<a name="agwf-maintenance-on-28-october-2026"></a>

# Maintenance d’AGWF du 28 octobre 2026

> Compte rendu de maintenance historique. Les observations et autorisations ci-dessous appartiennent à cette intervention. Le [README](../README.md) décrit le fonctionnement actuel.

Chacune des six compétences de rôle a reçu son propre fichier `agents/openai.yaml` avec `allow_implicit_invocation: false`. Leurs descriptions s’appliquent aux tâches préparées par AGWF. Un rôle s’arrête s’il ne dispose pas de l’affectation et des preuves du modèle et de l’effort de raisonnement effectivement utilisés. Les compétences de rôle ne démarrent ni le flux de travail ni le moniteur. La compétence principale a supprimé une formulation laissant entendre que la sélection d’un projet précédent restait valable.

Le code source et l’installation ont été alignés sur les lectures partagées, le remplacement qui préserve les permissions et les nouvelles tentatives limitées pour les fichiers d’état sous Windows. `candidate.py` utilise le même lecteur compatible avec le partage pour les fichiers d’état qui évoluent. Si un lecteur externe verrouille le fichier, l’ancienne version reste en place. Un verrou persistant reste une erreur.

Les outils de contexte et de mesure ont été déplacés dans le dossier `tools/` du paquet. L’installation utilisateur ne dépend ainsi plus de la copie de travail du projet. Lors de ce contrôle historique, les 41 fichiers comparés correspondaient entre le code source, le paquet et la copie installée. L’ancienne copie de l’outil de contexte à la racine a été supprimée et les chemins dans les tests ont été mis à jour.

Le compte rendu signale 63 tests du paquet et de l’outillage, ainsi que neuf tests d’accès aux fichiers sous Windows. Les neuf tests Windows ont aussi été exécutés avec `-O` sur les fichiers installés. Les sept définitions de compétence et les empreintes des fichiers ont été vérifiées. Le test du paquet relatif aux six règles de rôle manquantes a échoué avant la correction, puis réussi après.

AGWF était désactivé pendant l’intervention. Les sept compétences sont restées désactivées. Les tests portaient sur les accès du moteur à des fichiers temporaires. Ils ne testaient pas la coexistence réelle avec Remote.

Les risques recensés à ce stade comprenaient le démarrage d’un second serveur Codex pour chaque relevé, le choix d’un autre CLI depuis PATH, la perte de la sortie d’erreur du serveur enfant et la vérification intégrale répétée du paquet. Cette première intervention n’a pas établi la cause du problème avec Remote. Le [compte rendu de suivi](agwf-tum-duzeltmeler-20261028.md) décrit les changements suivants. Le moniteur actuel ne démarre plus de second serveur Codex.
