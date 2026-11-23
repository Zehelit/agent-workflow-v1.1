<a name="install-verify-and-maintain-agwf"></a>

# Installer, vérifier et maintenir AGWF

Utilisez Python 3.11 ou ultérieur ainsi que Git. La construction et le candidat utilisent la bibliothèque standard de Python. Choisissez un répertoire de compétences où l’utilisateur prévu peut écrire. Les commandes ci-dessous utilisent PowerShell. Sous Linux ou macOS, sélectionnez le répertoire équivalent dans la racine Codex prévue et adaptez la syntaxe du shell. Les vérifications de liste de contrôle d’accès (ACL) sont propres à Windows.

<a name="build-from-the-selected-checkout"></a>

## Construire à partir de la copie de travail sélectionnée

Lancez ces commandes à la racine du dépôt AGWF. Avant d’accepter la construction, consignez le commit source et examinez les changements locaux. Le répertoire de sortie doit être nouveau.

```powershell
python --version
git rev-parse HEAD
git status --short
$build = Join-Path $env:TEMP ('agwf-build-' + [guid]::NewGuid().ToString('N'))
python -X utf8 -B tooling/build-candidate.py --output $build
if ($LASTEXITCODE -ne 0) { throw 'AGWF build failed.' }
$receipt = Get-Content -LiteralPath (Join-Path $build 'build-receipt.json') -Raw | ConvertFrom-Json
$pin = $receipt.manifest_sha256
$package = Join-Path $build 'agent-workflow-v1-1'
$entry = Join-Path $package 'scripts/candidate.py'
$manifest = Join-Path $package 'manifest.json'
```

Le constructeur produit un répertoire de paquet, un ZIP déterministe et un reçu de construction. Conservez le reçu accepté en dehors du paquet modifiable, avec la preuve du commit source. L’installateur attend la valeur `manifest_sha256` acceptée. Une empreinte recalculée après une modification inattendue constitue une autre observation. [Rattachement et installation du paquet](../skills/agent-workflow-v1-1/references/coexistence.md).

<a name="choose-a-human-or-agent-path"></a>

## Choisir une installation manuelle ou par agent

Pour la racine des compétences Codex de l’utilisateur courant, lancez les commandes suivantes. Pour un essai jetable, utilisez un répertoire isolé dont le nom se termine par `agent-workflow-v1-1`. La même commande sert à une première installation et à vérifier sans réécriture une installation identique.

```powershell
$codexRoot = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $env:USERPROFILE '.codex' }
$destination = Join-Path $codexRoot 'skills/agent-workflow-v1-1'
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination $destination
if ($LASTEXITCODE -ne 0) { throw 'AGWF installation failed.' }
```

`activated:false` indique que l’opération a installé ou vérifié les fichiers du paquet sans démarrer de flux de travail. Sous Windows, le répertoire temporaire hérite des droits du dossier parent de la destination. Vérifiez avec le compte utilisateur prévu, sans élévation, que les fichiers installés sont lisibles et exécutables.

Pour une installation par agent, procédez ainsi.

1. Confirmez la destination demandée et vérifiez s’il s’agit d’une première installation, d’une mise à jour ou d’un essai isolé. Recensez le paquet courant, les modifications de l’utilisateur et les processus appartenant à l’installation avant de toucher une destination existante.
2. Vérifiez l’identité de la copie de travail et la version de Python. Construisez dans un répertoire neuf et conservez l’empreinte de manifeste acceptée comme ci-dessus.
3. Utilisez l’installateur du candidat figé pour la destination demandée. Avant une mise à jour, arrêtez uniquement les anciens processus concernés qui vous appartiennent et vérifiez leur fin effective. Si l’installateur refuse l’opération, il laisse la destination existante disponible pour inspection.
4. Vérifiez le manifeste et le contenu installés, puis lancez le contrôle passif ci-dessous depuis le point d’entrée installé. Indiquez le commit source, l’empreinte du manifeste, la destination et l’état obtenu. Précisez si vous avez effectué un contrôle passif ou lancé explicitement un flux réel.

<a name="verify-the-installed-entrypoint"></a>

## Vérifier le point d’entrée installé

Utilisez un nouveau projet jetable et un nouvel identifiant d’exécution. Le contrôle écrit un rattachement dans ce projet, sans démarrer de processus ni activer d’objectif.

```powershell
$probe = Join-Path $env:TEMP ('agwf-probe-' + [guid]::NewGuid().ToString('N'))
New-Item -ItemType Directory -Path $probe -ErrorAction Stop | Out-Null
$installedEntry = Join-Path $destination 'scripts/candidate.py'
$installedManifest = Join-Path $destination 'manifest.json'
python -X utf8 -B $installedEntry --manifest $installedManifest --manifest-sha256 $pin --version 1.1 --project $probe --run-id install-check workflow validate
if ($LASTEXITCODE -ne 0) { throw 'AGWF binding validation failed.' }
```

Contrôlez `identity`, `version`, `run_id`, `manifest_sha256` et `missing_dependencies` dans la réponse. Lors d’un essai isolé pour ce guide, l’installateur a indiqué `activated:false`. La validation a créé `run-binding.json` sans fichier d’état du flux ni répertoire de surveillance. Une empreinte de manifeste erronée a été refusée avec le code de sortie 2.

L’invocation réelle dans Codex intervient séparément. La session déjà ouverte peut nécessiter une actualisation du catalogue des compétences. Sélectionnez explicitement `$agent-workflow-v1-1` et indiquez le périmètre de la tâche. Le [guide d’exploitation](../skills/agent-workflow-v1-1/references/operations.md) couvre le démarrage autorisé et la vérification de l’état du moniteur.

<a name="update-recover-or-remove"></a>

## Mettre à jour, récupérer ou retirer une installation

- Pour mettre à jour une installation gérée par l’outil, arrêtez le processus concerné via son point d’entrée figé existant et vérifiez sa fin. Construisez et acceptez séparément le nouveau paquet. Ajoutez `--update` à la commande d’installation uniquement si l’installation appartient à l’outil et n’a pas été modifiée. Résolvez d’abord les modifications de l’utilisateur.
- L’installateur vérifie le paquet temporaire avant le remplacement et tente de restaurer le dossier précédent si la mise en place échoue. Gardez le dossier de transaction si la restauration échoue à son tour. Une mise à jour réussie ne conserve pas d’archive de restauration permanente. [Détails de l’installateur](../skills/agent-workflow-v1-1/references/coexistence.md).
- Une identité technique différente exige une nouvelle installation dans un nouveau répertoire. Conservez les anciens rattachements avec leurs manifestes d’origine et réglez séparément la découverte de la compétence installée. Ouvrez une nouvelle exécution après avoir accepté une mise à jour. [Limites de la migration](../skills/agent-workflow-v1-1/references/migration.md).
- Il n’existe pas de commande de désinstallation. Avant un retrait, confirmez le répertoire exact de la compétence, arrêtez les processus qui appartiennent à cette installation et conservez une copie vérifiée en dehors des répertoires de découverte. Préservez les projets utilisateur, leur état d’exécution, les points de reprise et les reçus externes. Vérifiez la découverte des compétences après le changement.
