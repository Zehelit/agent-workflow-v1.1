<a name="install-verify-and-maintain-agwf"></a>

# Installare, verificare e mantenere AGWF

Usa Python 3.11 o successivo e Git. La build e il candidato usano la libreria standard di Python. Installa in una directory delle skill in cui l’utente previsto può scrivere. I comandi seguenti usano PowerShell. Su Linux o macOS scegli la directory corrispondente sotto la radice Codex prevista e adatta la sintassi della shell. I controlli delle liste di accesso (ACL) sono specifici di Windows.

<a name="build-from-the-selected-checkout"></a>

## Costruire dal checkout selezionato

Esegui questi comandi dalla radice del repository AGWF. Registra il commit sorgente ed esamina le modifiche locali prima di accettare la build. La directory di output deve essere nuova.

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

La build produce una directory del pacchetto, uno ZIP deterministico e una ricevuta di build. Conserva la ricevuta accettata fuori dal pacchetto modificabile insieme al riscontro del commit sorgente. L’installatore richiede il valore `manifest_sha256` approvato. Un hash ricalcolato dopo una modifica inattesa è un’osservazione diversa. [Associazione al pacchetto e installazione](../skills/agent-workflow-v1-1/references/coexistence.md).

<a name="choose-a-human-or-agent-path"></a>

## Installare a mano o tramite un agente

Per usare la radice delle skill Codex dell’utente corrente, esegui i comandi seguenti. Per un test temporaneo scegli una directory isolata che termini con `agent-workflow-v1-1`. Lo stesso comando serve per una prima installazione e per controllarne una identica senza riscriverla.

```powershell
$codexRoot = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $env:USERPROFILE '.codex' }
$destination = Join-Path $codexRoot 'skills/agent-workflow-v1-1'
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination $destination
if ($LASTEXITCODE -ne 0) { throw 'AGWF installation failed.' }
```

`activated:false` indica che sono stati installati o controllati i file del pacchetto senza avviare un flusso. In Windows la directory di preparazione eredita l’accesso dalla directory superiore della destinazione. Verifica che il normale utente previsto possa leggere ed eseguire i file installati.

Se incarichi un agente dell’installazione, procedi così.

1. Conferma la destinazione e stabilisci se si tratta di una prima installazione, di un aggiornamento o di una prova isolata. Prima di modificare una destinazione esistente, fai l’inventario del pacchetto, delle modifiche utente e dei processi di competenza.
2. Controlla identità del checkout e versione di Python. Costruisci in una nuova directory e conserva l’hash del manifest approvato come descritto sopra.
3. Usa l’installatore del candidato fissato per la destinazione richiesta. Prima di aggiornare, ferma solo i vecchi processi pertinenti che appartengono all’installazione e verifica che siano usciti. Se l’installatore rifiuta l’operazione, lascia la destinazione esistente disponibile per l’esame.
4. Verifica il manifest e i file installati, poi esegui il controllo passivo qui sotto dal punto di ingresso installato. Riporta commit sorgente, hash del manifest, destinazione, stato ottenuto e se si è trattato di un controllo passivo o di un flusso effettivamente avviato su richiesta esplicita.

<a name="verify-the-installed-entrypoint"></a>

## Verificare il punto di ingresso installato

Usa un nuovo progetto temporaneo e un nuovo identificativo di esecuzione. Il controllo scrive un’associazione nel progetto temporaneo senza avviare processi o attivare obiettivi.

```powershell
$probe = Join-Path $env:TEMP ('agwf-probe-' + [guid]::NewGuid().ToString('N'))
New-Item -ItemType Directory -Path $probe -ErrorAction Stop | Out-Null
$installedEntry = Join-Path $destination 'scripts/candidate.py'
$installedManifest = Join-Path $destination 'manifest.json'
python -X utf8 -B $installedEntry --manifest $installedManifest --manifest-sha256 $pin --version 1.1 --project $probe --run-id install-check workflow validate
if ($LASTEXITCODE -ne 0) { throw 'AGWF binding validation failed.' }
```

Controlla `identity`, `version`, `run_id`, `manifest_sha256` e `missing_dependencies` nella risposta. In una prova isolata per questa documentazione l’installazione ha restituito `activated:false`. La convalida ha creato `run-binding.json` senza file di stato del flusso o directory del monitor. Un hash di manifest errato è stato rifiutato con codice di uscita 2.

La successiva invocazione effettiva in Codex è un’operazione a sé. La sessione già aperta potrebbe richiedere un aggiornamento del catalogo delle skill. Seleziona esplicitamente `$agent-workflow-v1-1` e indica l’ambito del lavoro. La [guida operativa](../skills/agent-workflow-v1-1/references/operations.md) descrive l’avvio autorizzato e la verifica dello stato del monitor.

<a name="update-recover-or-remove"></a>

## Aggiornare, ripristinare o rimuovere

- Prima di aggiornare un’installazione gestita, ferma il processo interessato tramite il punto di ingresso fissato di quella installazione e verificane l’uscita. Costruisci e approva separatamente il nuovo pacchetto. Aggiungi `--update` al comando di installazione solo se l’installazione è gestita e invariata. Risolvi prima le modifiche dell’utente.
- L’installatore verifica il pacchetto preparato prima di sostituire il precedente. Se la messa in opera fallisce cerca di ripristinare la directory precedente. Se fallisce anche il ripristino, conserva la directory della transazione. Dopo un aggiornamento riuscito non rimane un archivio permanente di rollback. [Dettagli dell’installatore](../skills/agent-workflow-v1-1/references/coexistence.md).
- Un’identità tecnica diversa richiede un’installazione da zero nella nuova directory. Conserva le vecchie associazioni con i manifest originali e gestisci separatamente la configurazione dei percorsi di ricerca delle skill. Apri una nuova esecuzione dopo un aggiornamento approvato. [Limiti della migrazione](../skills/agent-workflow-v1-1/references/migration.md).
- Non esiste un comando di disinstallazione. Prima di rimuovere un’installazione, conferma l’esatta directory della skill, ferma i processi che le appartengono e conserva una copia verificata fuori dai percorsi di ricerca delle skill. Preserva progetti dell’utente, stato delle esecuzioni, punti di ripristino e ricevute esterne. Dopo la modifica verifica quali skill sono visibili.
