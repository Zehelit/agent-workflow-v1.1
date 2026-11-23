<a name="pinned-runtime-binding-and-installation"></a>

# Associazione dell’esecuzione e installazione con hash fissato

L’identità tecnica è `agent-workflow-v1-1` e il nome visualizzato è `agent-workflow-v1.1`. L’invocazione implicita è disattivata. Questo candidato si adotta soltanto con una selezione esplicita. Il modello del client principale rimane una scelta dell’utente. Per i ruoli del flusso sono ammessi soltanto `gpt-6.1-sol`, `gpt-6-astra` o `gpt-6.1-terra` con livello di ragionamento almeno `medium`. Il ruolo interno di monitoraggio usa Terra/medium. Non sono previsti ripieghi.

Con una selezione esplicita attuale, l’agente principale deve avviare e verificare nello stesso turno l’esecuzione circoscritta. La selezione può riattivare quell’esecuzione dopo una precedente disattivazione. Le vecchie selezioni e la manutenzione del pacchetto non bastano. Se mancano i riscontri di attivazione completa o è vietato creare un obiettivo nativo, avvia manualmente il monitor come descritto nella [guida operativa](operations.md). Prima di iniziare il lavoro effettivo, verifica l’avvio. Non sostituire in silenzio il monitoraggio con lavoro diretto. Eventuali istruzioni di arresto successive hanno priorità.

Costruisci con Python 3.11 o successivo e la libreria standard.

```powershell
python -X utf8 -B tooling/build-candidate.py --output C:/staging/build-one
```

L’output contiene `agent-workflow-v1-1/`, uno ZIP deterministico e `build-receipt.json`. Conserva il valore `manifest_sha256` della ricevuta con i riscontri approvati di coordinamento o passaggio, fuori dal pacchetto modificabile. L’hash fornito dal chiamante è il riferimento di fiducia. Il controllo completo di manifest e dipendenze precede l’associazione, le modifiche rilevanti allo stato e i passaggi di coordinamento. Si effettua anche a ogni ciclo di misura del monitor. Il segnale periodico controlla solo i piccoli dati di esecuzione ed epoca. Non attraversa e non calcola l’hash del pacchetto ogni secondo. Ricalcolare lo SHA di un manifest cambiato e passare il valore nuovo non stabilisce un’attestazione esterna. Python verifica integrità dei byte e identità. Non autentica l’approvazione umana, i metadati di fornitore o modello o il file system controllato dal chiamante. Un interprete o eseguibile compromesso, o un chiamante che modifica sia l’associazione sia l’hash esterno approvato, esula da queste garanzie.

Le opzioni globali precedono la modalità. Dal sorgente usa `skills/agent-workflow-v1-1/scripts/candidate.py` con il manifest prodotto dalla build. Dall’installazione usa `scripts/candidate.py` e il manifest installati, con lo stesso hash esterno conservato. Ogni associazione da sorgente o installazione usa un ID d’esecuzione distinto. L’associazione immutabile comprende percorsi risolti delle dipendenze e del manifest, progetto, esecuzione, versione e hash. Spostare un’esecuzione su un altro punto di ingresso, sostituire il pacchetto, modificare o far mancare le dipendenze o riusare l’ID con un altro hash fa rifiutare l’operazione. Dopo un aggiornamento approvato avvia una nuova esecuzione.

```powershell
$pin = '<preserved manifest_sha256 from accepted build receipt>'
$entry = 'C:/staging/build-one/agent-workflow-v1-1/scripts/candidate.py'
$manifest = 'C:/staging/build-one/agent-workflow-v1-1/manifest.json'
$common = @('-X','utf8','-B',$entry,'--manifest',$manifest,'--manifest-sha256',$pin,'--version','1.1','--project','C:/project','--run-id','candidate-run-1')
python @common workflow validate
python @common workflow status
python @common coordination --input C:/project/batch.json
```

Un `workflow activate` o `bind-active-goal` autorizzato avvia il monitor e verifica il primo campione valido trasmesso con `--usage-input`. Dopo una continuazione valida o il recupero verificato di una misura scaduta durante un obiettivo attivo già autorizzato, `resume` apre una nuova epoca con una nuova ricevuta nativa. Vedi la [guida operativa](operations.md) per le procedure di ripresa e i [rilevamenti nativi](native-usage.md) per le misurazioni. Per misurare non si avviano server o CLI di Codex. I comandi attivi rinnovano il lease. `complete` e `halt` fermano il processo Python di competenza e ne verificano l’uscita. Anche i comandi espliciti del `watcher` seguono il contratto delle ricevute.

`workflow` accetta i comandi tipizzati del motore e `validate`. I comandi leggono JSON da `--input` o dallo standard input. Le vecchie CLI del motore `native/install/export/migrate` non sono supportate dal candidato. L’autorizzazione per `activate` e i riscontri per `bind-active-goal` devono riportare l’ID d’esecuzione selezionato. `validate` controlla ruoli e dipendenze e associa lo spazio di esecuzione, senza attivare l’obiettivo. Un lotto ha la forma `{ "batch_id": "inspect", "steps": [{ "key": "status", "command": "status", "payload": {} }] }`. Ha al massimo 64 passaggi, un solo autore principale, risultati in attesa registrati in modo persistente e nessun nuovo tentativo o invio nativo automatico. Per i comandi di lavoro restano indispensabili autorizzazione tipizzata attuale e riscontri di accettazione. Lo stato terminale nell’inventario non accetta il risultato. Le attività obsolete i cui autori hanno smesso di scrivere si possono scartare con `finish accepted:false`. Non si può fare mentre scrivono. La compattazione richiede un punto di ripristino verificato e archivia lo storico di attività e invalidazioni.

La radice isolata è `<project>/.agent-workflow-v1-1/runs/<run-id>/`. Le radici di stato operativo sono riservate e non si possono assegnare in scrittura a un’attività. Il monitor integrato usa `watcher/` in quella radice, uno strumento esplicito di pianificazione fissato, dati di appartenenza immutabili e gli argomenti per riavviare il punto di ingresso del candidato. Controlli riferiti a un’altra esecuzione o l’uso diretto della CLI privata non possono rinnovare il lease o fermare il processo altrui. Una chat inattiva non rinnova il lease a durata limitata. Usa un comando attivo del candidato nei passaggi chiave. Gli aggiornamenti non migrano in silenzio i processi in esecuzione. Il monitor non ha una skill o un programma di avvio separati. Le dipendenze private sono caricate dal pacchetto approvato.

Una prima installazione, un controllo senza modifiche e un aggiornamento gestito richiedono una destinazione esplicita che termini con `agent-workflow-v1-1`.

```powershell
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination C:/staging/skills/agent-workflow-v1-1
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination C:/staging/skills/agent-workflow-v1-1 --update
```

Gli aggiornamenti richiedono un’installazione gestita con byte invariati. Il programma verifica il pacchetto preparato prima di sostituire la directory esistente. Se la sostituzione fallisce, ripristina la directory precedente. Se riesce, rimuove i dati della transazione. Non fornisce un archivio permanente o un comando per tornare indietro. Se fallisce anche il ripristino, la directory della transazione conserva l’unica copia precedente per il recupero esplicito. Non eliminarla prima di avere verificato il recupero. Questi comandi preparano soltanto i file, come indica `activated:false`. La presenza nel catalogo, la selezione nativa esplicita, i metadati sul modello effettivamente eseguito dal fornitore e la prova reale del prodotto o del caso pilota si verificano dopo l’installazione.

In Windows l’installatore reimposta i controlli di accesso della directory temporanea affinché ereditino i permessi dalla directory superiore della destinazione, prima di copiare il pacchetto. Non indica un account Windows specifico. Un altro utente, o uno rinominato, riceve i diritti già assegnati alla sua directory superiore `skills`. Questa deve concedere lettura ed esecuzione all’utente previsto. Dopo l’installazione verifica sia che Codex trovi la skill, sia che il candidato si esegua direttamente in una normale sessione senza privilegi elevati. Il successo in una sessione elevata non è sufficiente.
