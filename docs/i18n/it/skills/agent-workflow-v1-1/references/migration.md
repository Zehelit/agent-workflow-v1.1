<a name="state-compatibility-and-recovery-boundaries"></a>

# Compatibilità dello stato e limiti del ripristino

L’ambiente di esecuzione v1.1 accetta solo lo stato corrente schema2 in uno spazio di esecuzione approvato e immutabile. Uno stato sconosciuto, danneggiato o obsoleto viene rifiutato senza modificare lo stato operativo. Vecchi testi di autorizzazione, ricevute di attività archiviate e osservazioni scadute sull’obiettivo non conferiscono autorizzazioni.

Il punto di ingresso fissato `scripts/candidate.py` non offre un comando di migrazione. Il motore non ha un convertitore di vecchi stati o uno strumento di convalida per le vecchie autorizzazioni. Non sono supportati l’importazione automatica, l’avvio basato su un’altra installazione o il ritorno al suo pacchetto. Avvia una nuova esecuzione dopo un aggiornamento approvato. Il manifest e l’associazione al programma ausiliario dell’esecuzione corrente devono restare invariati.

Il ripristino corrente usa il punto di ripristino verificato dell’esecuzione accettata e una nuova autorizzazione valida dell’utente. Conserva gli orari delle fonti, gli identificativi riservati contro il riuso, l’integrità dei file e l’osservazione che gli agenti con accesso in scrittura si sono fermati. I vecchi testi di arresto o continuazione restano riscontri storici. Il programma ausiliario non attiva automaticamente l’obiettivo ripristinato.

I nuovi punti di ripristino appartengono a `agent-workflow-v1-1`. La directory di stato predefinita del motore usato da solo è `.agent-workflow-v1-1`. Il punto di ingresso supportato e fissato usa `.agent-workflow-v1-1/runs/<run-id>`. Le precedenti directory `.agent-workflow` e `.agent-workflow-v1-0` non vengono importate. Restano riservate e non possono essere percorsi di attività o destinazioni di ripristino. La pulizia delle istantanee rispetta i marcatori di appartenenza di queste directory. L’invocazione diretta della CLI del motore viene rifiutata.

<a name="move-the-user-installation-to-the-11-technical-identity"></a>

## Passare l’installazione utente all’identità tecnica 1.1

1. Fai l’inventario dei processi e degli agenti che scrivono. Ferma soltanto le attività di tua competenza. Prima di spostare un’installazione precedente, verifica l’arresto degli elementi di tua competenza e l’uscita effettiva dei processi.
2. Verifica il vecchio pacchetto rispetto al suo registro di appartenenza. Controlla gli hash di un backup collocato fuori da tutte le directory in cui si cercano le skill. Preserva eventuali modifiche prima di proseguire.
3. Installa da zero il nuovo pacchetto in `skills/agent-workflow-v1-1`, con il nuovo manifest e l’hash conservato all’esterno. Non usare `--update` per un’identità tecnica precedente.
4. Prima di cambiare i percorsi di ricerca, esegui `workflow validate` dal nuovo punto di ingresso installato, con un nuovo identificativo univoco di esecuzione per il controllo. Confronta identità, versione, area di lavoro e hash del manifest restituiti. Questa verifica crea `run-binding.json` senza avviare obiettivo, stato del flusso o monitor.
5. Il nuovo comando d’invocazione è `$agent-workflow-v1-1`. Dopo la verifica diretta, aggiorna in Codex i percorsi della skill principale e di ruolo, conservando le impostazioni di attivazione e di invocazione esplicita. Proteggi le modifiche di configurazione concorrenti. Ricontrolla i processi di tua competenza e preserva eventuali installazioni attive usate da altre sessioni. Dopo il passaggio, archivia il vecchio pacchetto fuori dai percorsi di ricerca delle skill, poi verifica configurazione e backup.
6. Mantieni identità, percorsi e hash dei vecchi punti di ripristino, associazioni e ricevute. Apri una nuova esecuzione con il nuovo pacchetto. Se resta lavoro rilevante da finire, fornisci alla nuova esecuzione separatamente i risultati precedenti verificati e l’autorizzazione attuale effettiva.
