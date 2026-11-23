<a name="changelog"></a>

# Historique des modifications

Ce document retrace les versions existantes. Le code source extrait et le reçu de construction accepté permettent d’identifier précisément le paquet.

<a name="11--2026-11-10"></a>

## 1.1 — 10 novembre 2026

<a name="added"></a>

### Ajouts

- Trace durable du premier essai avec activate ou bind-active-goal. Pour réessayer en toute sécurité, conservez l’empreinte SHA-256 du reçu d’échec en dehors du paquet.
- Verrou en écriture détenu par l’exécution pour sérialiser les opérations de cycle de vie du candidat.
- tooling/run-lifecycle-check.py pour la préparation, le démarrage, l’actualisation des observations, la reprise après arrêt et la clôture normale.
- Mode timed-test explicite, avec des horodatages distincts pour SAVE, le point de reprise et l’arrêt, et un classement des événements tardifs. Cet outil ne fournit aucune preuve d’une mise en pause native.
- Heure réelle du relevé, échéance, intervalle et suggestion de mise à jour anticipée dans le résumé d’état.
- Présentation du projet, diagrammes d’architecture, de flux de travail et d’orchestration, instructions d’installation manuelle et par agent, et guide du cycle de vie.

<a name="fixed"></a>

### Corrections

- Reprise contrôlée sous le même objectif actif et autorisé lorsque le relevé d’utilisation expire. Un quota réellement épuisé ou un arrêt demandé par l’utilisateur suit sa propre procédure.
- Le nouvel essai après un premier démarrage raté conserve le périmètre, la trace du SAVE antérieur et le rattachement au paquet. Il exige toujours la vérification de l’absence du processus.
- Une exception limitée dans le temps pour la fenêtre hebdomadaire s’applique de façon identique dans le moteur et dans le moniteur, en conservant les valeurs observées.
- Des tentatives en nombre limité traitent les erreurs temporaires de partage et de lecture des verrous Windows lorsqu’elles sont reconnues. Les erreurs permanentes et non classées restent visibles.
- Précisions sur l’actualisation des mesures, le renouvellement du bail et la fraîcheur des preuves après arrêt.
- Mise à jour des anciennes signatures d’appel des utilitaires de test et des attentes liées aux fichiers et aux processus.

<a name="compatibility-and-evidence"></a>

### Compatibilité et preuves

- Le nom d’appel, le répertoire d’installation et l’espace de noms d’exécution passent ensemble en 1.1. Ils deviennent `$agent-workflow-v1-1`, `agent-workflow-v1-1/` et `.agent-workflow-v1-1/runs/`.
- Les nouveaux manifestes refusent l’ancienne identité technique. Les exécutions précédentes restent des preuves historiques et ne peuvent pas être rattachées automatiquement. Leurs espaces de noms restent réservés pour établir la propriété et restaurer les données.
- Une mise à jour ne réécrit pas le rattachement immuable d’une exécution à son paquet. Ouvrez une nouvelle exécution après avoir accepté le nouveau paquet.
- Python 3.11 ou ultérieur et bibliothèque standard uniquement. Le moniteur ne peut pas appeler directement les outils natifs de mesure et ne lance pas de second serveur Codex.
- Les tests isolés utilisent des observations synthétiques de l’hôte et de vrais processus Python. L’interface Remote, l’action Continue et la mesure du temps côté interface exigent des preuves distinctes en conditions réelles.
- Cette version a été consignée dans des commits source. La publication d’une version GitHub et la création d’un tag étaient hors du périmètre consigné.

<a name="10--previous-source-version"></a>

## 1.0 — version source précédente

- Point d’entrée du candidat rattaché à une empreinte de manifeste, état de travail typé, documents de rôle et contrôles des droits d’écriture et de capacité.
- Moniteur intégré sans modèle, points de reprise et restauration, journal de coordination durable et construction déterministe du paquet.
- L’historique ne fait aucune affirmation sur l’existence d’une version GitHub ou d’un tag correspondant.
