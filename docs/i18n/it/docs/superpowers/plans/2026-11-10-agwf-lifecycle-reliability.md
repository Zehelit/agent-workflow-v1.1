<a name="lifecycle-startup-and-recovery-implementation-plan"></a>

# Piano di implementazione per avvio e ripresa

> Piano storico del 10 novembre 2026. Gli elenchi descrivono i lavori proposti all’epoca. Le autorizzazioni e gli artefatti privati citati riguardano quell’intervento. La [guida operativa](../../../skills/agent-workflow-v1-1/references/operations.md) descrive il comportamento attuale.

<a name="goal-and-constraints"></a>

## Obiettivo e vincoli

Il piano prevedeva un nuovo tentativo sicuro dopo un primo avvio fallito, aggiornamento puntuale delle misure, ripresa verificata e chiusura ordinaria. Manteneva il candidato fissato, il motore tipizzato e il monitor senza modello. Lo strumento di verifica avrebbe distinto preparazione, osservazione effettiva dell’host, esecuzione e chiusura. La prova temporizzata sarebbe rimasta una modalità esplicita. Vedi la [specifica storica](../specs/2026-11-10-agwf-lifecycle-reliability.md).

Il piano originario autorizzava il commit e il push del sorgente v1.1 pertinente. Escludeva la creazione di una release GitHub e del tag di versione. Richiedeva Python 3.11 o successivo, libreria standard e unittest, gli strumenti Codex esistenti per obiettivo e utilizzo e il comportamento di Windows per processi e condivisione file. Preservava il lavoro in corso, i riscontri delle esecuzioni precedenti e le associazioni ai pacchetti. Richiedeva di osservare pacchetto, obiettivo, misurazione, citazioni dell’utente e identità effettiva dei processi. I dati sintetici erano riservati ai test isolati. Scadenza del rilevamento, quota effettiva, riserva anticipata e arresti utente avevano requisiti distinti di autorizzazione. Il funzionamento normale non prevedeva un SAVE artificiale. Restavano fuori dall’intervento il lavoro di altre sessioni e le impostazioni del modello scelte dall’utente.

<a name="review-focus"></a>

## Punti da verificare

1. Un SAVE reale per scadenza della misura dopo un primo avvio fallito dovrebbe consentire un nuovo tentativo se ne ricorrono le condizioni, anche prima che esista lo stato del programma ausiliario.
2. Quota, arresto utente, nuovo obiettivo o processo sconosciuto non devono superare il controllo di ripresa tecnica.
3. Un avvio parziale o concorrente resta incerto finché riscontri supportati non chiariscono l’esito.
4. Un file di rilevamento più recente non cambia l’ora della sua osservazione. Una misura recente può coesistere con un lease scaduto.
5. Un evento nativo tardivo non supera il controllo temporale. Un errore temporaneo di condivisione in Windows non prova l’uscita del processo.

<a name="files-and-responsibilities"></a>

## File e responsabilità

| Percorso | Responsabilità prevista |
| --- | --- |
| `skills/agent-workflow-v1-1/scripts/candidate.py` | Tentativo di avvio, decisione di ritentare, appartenenza del monitor ed esiti del ciclo di vita |
| `skills/agent-workflow-v1-1/private/limit-watcher/scripts/monitor.py` | Comportamento di SAVE, epoca e ricevuta, con modifiche limitate a quanto serve al contratto |
| `skills/agent-workflow-v1-1/scripts/workflow.py` | Regole esistenti per attivo, arrestato e completato e per l’arresto dell’utente |
| `tooling/run-lifecycle-check.py` | Strumento di verifica preparato, alimentato da riscontri nativi dell’agente principale |
| `tooling/tests/test_lifecycle_check.py` | Test di preparazione, eventi tardivi e classificazione dei risultati |
| `skills/agent-workflow-v1-1/tests/test_candidate_coexistence.py` | Integrazione del pacchetto sorgente e installato con veri processi Python di competenza |
| `test_native_usage_source.py`, `test_goal_binding.py`, `test_watcher_engine.py`, `test_monitor.py` e `test_watcher_atomic_write.py` | Test di limiti delle ricevute, obiettivi e autorizzazioni, tempi del monitor e comportamento Windows |
| `SKILL.md`, `references/operations.md`, `references/native-usage.md` e `references/coexistence.md` | Istruzioni per aggiornamento, nuovo tentativo, installazione e accettazione separata sull’host |

<a name="task-1-retry-a-failed-first-activation"></a>

## Attività 1. Riprovare dopo una prima attivazione fallita

Le interfacce previste comprendevano `activation_identity(command: str, payload: dict) -> dict`, `record_activation_attempt(runtime, record: dict) -> None` e `startup_retry_options(runtime, engine, command: str, payload: dict) -> tuple[str, ...]`. Il file `activation-attempt.json` per esecuzione registrava identità della fonte, `schema_version`, `command`, `pre_state_sha256`, `phase` e dettagli osservati del monitor e dell’errore. `prepared`, `failed` e `committed` sono fasi distinte. Una voce `prepared` rimasta incompleta va riconciliata con lo stato reale.

- Dimostrare prima il fallimento con una ricevuta vecchia, poi riprovare con una nuova misura per lo stesso obiettivo sia con `bind-active-goal` sia con `activate`. Preservare l’hash del SAVE originario e l’autorizzazione utente. Verificare un processo funzionante e lo stato attivo.
- Salvare l’intenzione di avvio dopo `preflight_activation`. Legare comando, esecuzione, obiettivo nativo, chat, ambito, testo dell’obiettivo e autorizzazione. Il solo cambio dell’orario di osservazione non crea un nuovo ambito.
- Usare, tra avvio e passaggi del motore, un blocco del ciclo di vita appartenente all’esecuzione e riservato a un solo autore. Rifiutare un blocco di proprietario sconosciuto. Interfaccia nativa e stato Python si aggiornano con operazioni distinte.
- Usare la nuova epoca tecnica solo per un `usage_receipt_expired` accertato nello stesso ambito, con nuova osservazione attiva e rilevamento reale. Un flag del chiamante non conferisce l’autorizzazione. Registrare il successo dopo avere verificato il processo e confermato lo stato del motore.
- Provare i rifiuti per quota, riserva anticipata, altra diagnosi, nuovo SAVE, arresto utente, obiettivo diverso o inattivo, processo attivo o sconosciuto, registro modificato o incompleto, pacchetto errato e avvi concorrenti. Provare la ripresa dopo una ricevuta di tentativo non valida. Il ritorno allo stato precedente ripristina soltanto i controlli verificati che rientrano nel perimetro dell’errore e salva eventuali SAVE più recenti.
- Eseguire i test di convivenza del candidato e dei rilevamenti nativi. Controllare stato attivo, identità attuale e validità, storico conservato e uscita effettiva del processo.

<a name="task-2-prepare-before-measuring-and-refresh-early"></a>

## Attività 2. Preparare prima di misurare e aggiornare in anticipo

Lo strumento avrebbe offerto `prepare`, `run --input PATH`, `observe --input PATH` e `close`. La preparazione verifica pacchetto, hash, ruoli e comandi prima di avviare. `refresh_by(at: float, interval: float) -> float` suggerisce `at + interval - min(15, interval / 4)`. Con un intervallo di 60 secondi, suggerisce l’aggiornamento a 45 secondi. La scadenza effettiva non cambia.

- Verificare che preparazione e aiuto non richiedano input reale, che input vecchi o mancanti non creino stato attivo e che ripresentare un rilevamento non cambi l’ora di osservazione.
- Acquisire obiettivo e misura nativi dopo la preparazione. Copiare solo campi approvati, comprese entrambe le finestre reali e gli orari di azzeramento. Escludere account, identità e dati di credito.
- Distinguere `sample_at`, `next_check_at`, `check_interval_seconds` e `refresh_by`. La pubblicazione di una ricevuta o un segnale periodico riuscito non dimostrano l’attualità del monitoraggio.
- Richiedere `status.at == usage.observedAt`, identità del processo in esecuzione, epoca corrispondente, `sampleFresh` e `leaseValid`. Verificare il rinnovo dopo l’acquisizione. Non promettere aggiornamenti durante una chiamata non interrompibile.
- Tenere disattivata l’iniezione di timer nel lavoro ordinario. Controllare le istantanee se richiesto e confermare completamento e uscita del processo alla chiusura. Verificare che la chiusura ordinaria non abbia generato un blocco SAVE artificiale.
- Esercitare le suite del ciclo di vita e dei rilevamenti nativi con orari diversi di osservazione e pubblicazione.

<a name="task-3-diagnose-timing-without-confusing-late-events-with-success"></a>

## Attività 3. Diagnosticare i tempi riconoscendo gli eventi tardivi

- Usare `timed-test` solo su richiesta esplicita, con un processo preparato e funzionante, un evento reale e `--event-input PATH --delay-seconds 10 --tolerance-seconds 1`. La scadenza prevista era `event_at + 10` e l’attesa usa un orologio monotono.
- Un evento osservato con 45 secondi di ritardo deve restituire `INVALID_TIMING`. Le prove storiche da 98,74 e 45,05 secondi restano fallimenti.
- Usare i riscontri ufficiali esistenti sull’obiettivo. Se manca una fonte supportata di eventi nativi, classificare l’osservazione come tardiva o incompleta. Non usare trasporti privati o archivi di credenziali per misurare l’interfaccia.
- Registrare separatamente `native_event_at`, `observed_at`, `worker_healthy_at`, `save_requested_at`, `save_returned_at`, `snapshot_verified_at`, `helper_halted_at` e `native_paused_at`. La finestra utile di 9–11 secondi vale per la richiesta SAVE quando il processo funzionava già. La pausa nativa richiede riscontri propri.
- Nei due passaggi effettivi usare lo stesso obiettivo e la stessa esecuzione. Dimostrare stato attivo, punto di ripristino e arresto, poi verificare una vera continuazione idonea. Una prima associazione da stato intatto non dimostra la ripresa.
- Provare eventi sintetici in anticipo, in ritardo, ripetuti e modificati, mancanza di riscontri sulla pausa, arresto esplicito e i due passaggi. Tenere il controllo temporale dal vivo facoltativo e separato dall’accettazione ordinaria.

<a name="task-4-verify-recovery-and-regressions"></a>

## Attività 4. Verificare ripresa e regressioni

- Provare primo avvio, scadenza al primo tentativo e nuovo tentativo con misura recente, scadenza di un’esecuzione attiva seguita da arresto, istantanea, arresto definitivo e ripresa, tentativo invalido poi valido, continuazione nativa da pausa ad attivo e chiusura naturale. Usare i punti di ingresso pertinenti da sorgente e pacchetto.
- Prima di proseguire, osservare `halted` e verificare l’istantanea. Dopo la ripresa verificare obiettivo ed esecuzione originali, nuova epoca, hash SAVE conservato, autorizzazione e stato effettivo del processo.
- Rifiutare esaurimento quota, riserva anticipata, un nuovo SAVE per quota, arresto utente, obiettivo errato, ricevute scadute o ripetute, punti di ripristino corrotti e sottoagenti attivi con esito irrisolto. Lo stato nativo `blocked` mantiene i propri criteri.
- Nei test Windows copiare solo i file necessari appartenenti all’esecuzione e controllare l’identità del processo insieme allo stato finale. Gli errori temporanei di condivisione non implicano l’uscita. Gli errori permanenti di permesso devono emergere entro un intervallo limitato.
- Dopo i casi mirati eseguire entrambe le suite del repository. Il numero dei test si ricava dall’esecuzione. Il normale ciclo effettivo richiede un obiettivo reale e uno strumento preparato. Scadenza e diagnosi temporale richiedono autorizzazioni proprie e possono restare `not_tested`.

<a name="task-5-document-the-contract-and-verify-distribution"></a>

## Attività 5. Documentare il contratto e controllare la distribuzione

- Documentare il primo nuovo tentativo, la ripresa dopo arresto e i percorsi separati per quota e arresto utente. Spiegare l’ordine degli aggiornamenti della misura e del lease e cosa può dimostrare la diagnosi di un evento tardivo.
- Inventariare pacchetto e lavoro già esistente. Preservare modifiche e installazioni in uso. Non dedurre attività dall’appartenenza a un vecchio hash e non migrare esecuzioni attive.
- Costruire due volte dallo stesso sorgente. Confrontare hash di manifest e ZIP. Verificare i file modificati nella build e nei test di installazione. Controllare il candidato installato con un’esecuzione passiva temporanea, verificare hash e accesso Windows e riportare il carattere passivo.
- Registrare separatamente i riscontri di `startup_retry`, `receipt_refresh`, `halted_resume`, `native_continue`, `natural_complete`, `timed_test`, `worker_exit` e `package_match`. Conservare le prove negative precedenti.
- Verificare autorizzazione, ritorno allo stato precedente e conservazione del SAVE prima della pubblicazione di sorgenti autorizzata all’epoca. Lasciare fuori da Git rapporti privati, ricevute e dati di esecuzione.

<a name="task-6-align-the-existing-v11-identity-and-presentation"></a>

## Attività 6. Allineare identità e presentazione della v1.1 esistente

- Migrare insieme nome visibile, nome di invocazione, percorso del pacchetto e nuovo spazio dei dati d’esecuzione. Preservare identità precedenti e hash accettati.
- Riportare storia effettiva della v1.0 e modifiche della v1.1 in `CHANGELOG.md`. Spiegare nel README finalità, limiti, architettura, flusso di lavoro e orchestrazione.
- Descrivere installazione manuale e tramite agente da un checkout pulito, conservazione dell’hash, verifica e invocazione esplicita. La configurazione di per sé non avvia un obiettivo nativo e non modifica le impostazioni del modello.
- Per la pubblicazione autorizzata all’epoca, allineare About e Topics al sorgente. Preservare accessi del repository e protezione dei rami. Verificare i comandi del README e la corrispondenza dei diagrammi al sorgente.
- Pubblicare solo i file autorizzati, verificare commit e albero remoti, versione e documentazione e accertare che non siano stati creati release o tag.

<a name="acceptance-boundary"></a>

## Perimetro di accettazione

Il piano originario metteva le attività centrali da 1 a 5 prima del lavoro finale di pubblicazione. Richiedeva i casi di primo nuovo tentativo e di ripresa dopo un arresto effettivo, test di regressione, controlli sul pacchetto e osservazioni reali nel perimetro corretto. Il comportamento dell’interfaccia nativa richiedeva riscontri effettivi dell’host. Il risultato a dieci secondi era una verifica a parte, da svolgere su un evento nativo reale e con osservazione separata dei tempi.
