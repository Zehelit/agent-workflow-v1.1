<a name="command-contract-a02"></a>

# Specifica dei comandi (A02)

Il programma ausiliario, scritto con la libreria standard, valida i comandi che passano per `Workflow.execute` e `Workflow.apply`. Non intercetta gli strumenti nativi, non autentica le loro ricevute o i messaggi dell’utente, non ferma i processi e non osserva in modo autonomo i modelli effettivi. L’agente principale svolge queste operazioni e fornisce i riscontri. Restano validi i controlli A26 e quelli su punti di ripristino, appartenenza e capacità.

Lo stato usa l’intero `schema_version: 2`. Uno stato senza versione, obsoleto, futuro o malformato viene rifiutato prima di ogni modifica. Non esiste uno strumento di migrazione o importazione dei vecchi stati. Avvia una nuova esecuzione AGWF con una corrente autorizzazione tipizzata. Le istantanee e il ripristino dei punti restano supportati. I riscontri storici non autorizzano lavoro. Le prove con guasti simulati usano solo copie temporanee di test.

<a name="json-and-error-policy"></a>

## Regole per JSON ed errori

File di stato, file `--input`, standard input, metadati dei punti di ripristino e JSON di appartenenza dell’installatore usano lo stesso parser rigoroso. Le chiavi duplicate sono rifiutate a ogni profondità, anche se i valori coincidono. Sono rifiutati `NaN`, `Infinity`, `-Infinity` e i valori letterali che sembrano finiti ma eccedono il limite e diventano infiniti, come `1e999`. I file di input accettano UTF-8 con BOM facoltativo. La radice JSON di un comando deve essere un oggetto. Lo standard input vuoto fornisce `{}`, che è valido solo dove lo ammette il comando. `write_json` valida e serializza con `allow_nan=False` prima di creare directory o file temporanei.

Gli input sono oggetti JSON con chiavi stringa. Gli elenchi restano elenchi. Interi e valori in virgola mobile non sono booleani. Ogni indicatore presente deve avere esattamente il tipo Python `bool`, cioè `true` o `false` in JSON. Stringhe, 0/1, null, elenchi e oggetti non li sostituiscono. Omettere un indicatore obbligatorio fa rifiutare l’input. Quelli facoltativi sono false per impostazione predefinita soltanto se assenti. Null non è valido. I numeri devono essere finiti. I conteggi richiedono interi da 1 in su, gli orari accettano interi o valori in virgola mobile da 0 a sys.float_info.max e le percentuali vanno da 0 a 100 inclusi. Il limite superiore esclude interi enormi che eccederebbero la capacità dei calcoli temporali. Gli orari devono anche rispettare le regole di validità temporale del comando. Il limite massimo di capacità resta quello esistente. L’invio include l’agente principale nel conteggio dei posti e mantiene il limite dei sottoagenti.

Gli input vengono verificati prima della creazione di blocchi o directory e prima dell’esecuzione. `apply` lavora su una copia privata del JSON e la rende effettiva solo dopo il superamento dei controlli. Il rifiuto conserva byte e SHA-256 dello stato esistente, salvo la ricevuta persistente INPUT_INVALIDATED e gli indicatori contract_invalid descritti sotto. Un comando iniziale rifiutato non crea stato. Un’istantanea fallita può lasciare, come prima, una directory del punto di ripristino di competenza e non confermata. Un dato formalmente valido ma indisponibile porta alla transizione prevista descritta sotto.

Un errore del programma restituisce il codice di uscita 1 e invia questo oggetto su stderr.

```json
{"error":"exact boolean required","code":"FIELD_TYPE","field":"$.accepted"}
```

| Codice | Significato | Campo |
| --- | --- | --- |
| `COMMAND_UNKNOWN` | Comando sconosciuto del motore | `$.command` |
| `FIELD_REQUIRED` | Campo obbligatorio o opzione richiesta dell’utilità assente | Campo esatto mancante |
| `FIELD_UNKNOWN` | Campo sconosciuto o opzione CLI non applicabile al comando | Campo esatto contestato |
| `FIELD_TYPE` | Tipo errato fornito dal chiamante | Campo o indice esatto |
| `FIELD_VALUE` | Identificativo, testo o elenco obbligatorio vuoto, intervallo numerico o stato dell’inventario non valido | Campo o indice esatto |
| `JSON_DUPLICATE_KEY` | Membro duplicato nel JSON | Membro esatto, anche annidato |
| `JSON_NONFINITE` | Numero o token non finito, anche nei metadati grezzi | Campo o indice esatto |
| `JSON_SYNTAX` | Sintassi o codifica JSON invalida | `$` |
| `JSON_TYPE` | Valore non JSON o chiave di oggetto non testuale | Valore o oggetto esatto |
| `COMMAND_REFERENCE` | Attività indicata o campo memorizzato obbligatorio inesistente | `$.id` per la ricerca. Il messaggio indica la chiave mancante. |
| `COMMAND_CONSTRAINT` | Controllo su autorizzazione, stato, appartenenza, revisione o validità temporale | Campo pertinente se disponibile, altrimenti `$.command` per i controlli sullo stato intero |
| `STATE_VERSION` o `STATE_TYPE` | Versione non supportata o radice di stato diversa da un oggetto | `$.schema_version` o `$` |
| `IO_ERROR` o `INVALID_VALUE` | Errore del file system o altro errore relativo al valore | `$` |

I percorsi usano il punto per le chiavi semplici, le parentesi quadre per le altre e `[index]` per gli elenchi. Gli errori argparse ordinari, come opzioni o comandi sconosciuti o argomenti mancanti, mantengono stderr e codice 2 di argparse. Un’opzione riconosciuta ma non applicabile produce il suddetto errore strutturato `FIELD_UNKNOWN`. Nessuna modalità usa `assert`, quindi la convalida funziona anche con `python -O`.

<a name="payload-types"></a>

## Tipi di input

`text` è una stringa non vuota dopo avere eliminato gli spazi alle estremità ai fini della convalida. Il contenuto originale resta intatto. `string` può essere vuota. `task_id` corrisponde a `[a-z][a-z0-9_]{0,63}`. `strings` è un elenco di stringhe non vuote che può essere vuoto. `criteria` esige almeno un elemento. `task_ids` è un elenco di ID di attività. `timestamp`, `count` e `percent` seguono le regole numeriche precedenti. Solo il suffisso `?` permette null esplicito.

I comandi e gli oggetti strutturati annidati rifiutano campi fuori dall’elenco obbligatorio e facoltativo qui sotto. Due eccezioni conservano il risultato grezzo del fornitore. Il contenuto delle ricevute e i metadati estranei alla misura di `rateLimits` e finestre grezze rimangono JSON non interpretato. Rifiutano comunque chiavi duplicate e valori non finiti o non JSON. I metadati sconosciuti non si interpretano come indicatori o prove di misura valida.

| Tipo annidato | Campi obbligatori | Opzioni e vincoli |
| --- | --- | --- |
| `authorization` | `event_id`, `exact_quote`, `kind`, `source_kind`, `source_id`, `goal_id`, `thread_id`, `run_id` (text), `scope` (scope), `observed_at` e `source_created_at` (timestamp), `original_turn_id` e `observed_turn_id` (text) | Nessun altro campo. A26 controlla tipi consentiti, fonte utente diretta, ambito identico, riuso ed età. |
| `scope` | `kind`, `id`, `objective` (text) | `kind` è objective o report secondo A26. |
| `source` | `source_kind`, `source_id`, `exact_quote`, `original_turn_id` (text), `source_created_at` (timestamp) | Deve coincidere con i campi della fonte nell’autorizzazione fornita. |
| `native_binding` | `binding_id`, `goal_id`, `thread_id`, `run_id`, `objective` (text), `scope` (scope), `goal_created_at` e `observed_at` (timestamp) | Solo associazione iniziale N02. Ambito di tipo objective e testo esattamente coincidente. Creazione dell’obiettivo prima dell’osservazione. Nessun campo di fonte umana. |
| `app_goal` | `goal_id`, `thread_id`, `status`, `source_kind`, `source_id` (text), `observed_at` (timestamp) | `objective` (text) e `goal_created_at` (timestamp) facoltativi, entrambi obbligatori e uguali all’associazione se la base è nativa, anche a ogni aggiornamento. A26 richiede provenienza get_goal, identità coincidente, età massima di 60 secondi e stato attivo appropriato. |
| `inventory` | Elenco di oggetti con `id` e `status` (text) | Nessun campo extra. ID univoci e stati running, interrupting, unknown, idle, stopped e done. Osservazione da 0 a 60 secondi. |
| `models` | Oggetto che associa nomi di modello non vuoti a elenchi `strings` di livelli | Oggetto o elenco vuoto ammesso. prepare richiede sempre modello e livello esatti osservati. |
| `arguments` | Nessun campo strutturalmente obbligatorio | Solo `task_name` (task_id), `fork_turns` (text), `message` (string), `model` e `reasoning_effort` (text). Record esige uguaglianza completa con gli argomenti preparati. Campi mancanti impediscono l’invio. |
| `receipt` | Oggetto, elenco o stringa non vuoti | Intero contenuto JSON non interpretato dall’effettivo risultato dello strumento. Non ne attesta l’autenticità. |
| `rate_limits?` | Oggetto o null | primary e secondary facoltativi (window?). Altri metadati grezzi del fornitore non interpretati. |
| `window?` | Oggetto o null | `usedPercent` (percent?), `windowDurationMins` (count?) e `resetsAt` (timestamp?) facoltativi. Altri metadati non interpretati. |

<a name="engine-commands"></a>

## Comandi del motore

Gli elenchi di campi obbligatori e facoltativi sono esaustivi. Se non ci sono campi facoltativi, l’input ha solo quelli obbligatori. I comandi senza campi ricevono `{}`.

| Comando | Campi obbligatori | Campi facoltativi e impostazioni predefinite |
| --- | --- | --- |
| `activate` | `objective` (text), `authorization` (authorization), `interpretation` (string), `source_message` (source), `app_goal_observation` (app_goal), `models` (models), `slot_limit` (count), `inventory` (inventory), `observed_at` (timestamp) | `session_limit` (count?). Se assente o null, nessun limite aggiuntivo alle sessioni aperte. |
| `bind-active-goal` | `objective` (text), `native_binding` (native_binding), `interpretation` e `procedural_check` (text), `app_goal_observation` (app_goal), `models` (models), `slot_limit` (count), `inventory` (inventory), `observed_at` (timestamp) | `session_limit` (count?). Solo lo stato schema2 iniziale, inattivo, senza attività, campi o storico aggiuntivi. |
| `observe-authorization` | `authorization` (authorization), `interpretation` (string), `source_message` (source) | `app_goal_observation` (app_goal) ammessa, ma il comando non aggiorna l’osservazione conservata. |
| `app-goal` | `app_goal_observation` (app_goal) | Nessuno |
| `usage` | Nessuno | `observed_at` (timestamp), `rateLimits` (rate_limits?). L’omissione indica misura indisponibile. |
| `usage-due` | Nessuno | `substantial_work` (bool, false) |
| `inventory` | `inventory` (inventory), `observed_at` (timestamp) | Nessuno |
| `task` | `id` (task_id), `role` (text), `objective` (text), `criteria` (criteria) | `file_inputs` (file_inputs, []), `context_refs` (strings, []), `review_targets` (review_targets, []), `revision` (text, UUID generato se manca), `writes` (strings, []), `depends_on` (task_ids, []), `independent_review` (bool, false) |
| `prepare` | `id` (task_id) | `effort` (text, predefinito del ruolo se manca), `current_context` (context_pin per N06) |
| `record` | `id` (task_id), `submitted_arguments` (arguments), `session_id` (text), `receipt` (receipt) | `effective_model`, `effective_effort` (text?). Assenza o null indica dato non verificato. |
| `dispatch_failed` | `id` (task_id), `evidence` (text) | `certain_not_started` (bool, false). Assente o false lascia l’esito incerto. |
| `reconcile` | `id` (task_id), `evidence` (text), `no_owned_work_running` (bool) | Richiede true e inventario recente senza sottoagenti attivi. |
| `finish` | `id` (task_id), `accepted` (bool), `evidence` (text) | `review_task` (task_id), richiesto per accettare un’attività con revisione indipendente. |
| `cancel` | `id` (task_id), `reason` (text) | Solo un’attività pronta e mai inviata. |
| `stop` | Nessuno | Nessuno |
| `snapshot` | `writers_quiescent` (bool), `next_action` (text) | `files` (strings, []). L’indicatore obbligatorio deve valere true. |
| `restore` | `destination` (text) | Nuova destinazione esterna al progetto. |
| `halt` | `inventory` (inventory), `observed_at` (timestamp), `owned_processes_stopped` (bool), `stop_evidence` (text) | `degraded` (bool, false). Con true diventano obbligatori `reason` e `next_action` (text). L’indicatore di arresto deve valere true. |
| `resume` | `authorization` (authorization), `interpretation` (string), `source_message` (source), `app_goal_observation` (app_goal) | Nessuno. Occorrono continuazione corrispondente dopo l’arresto, misura e inventario recenti e punto di ripristino attuale verificato. |
| `compact` | Nessuno | Nessuno. Richiede attività terminali accettate o rifiutate, nessun sottoagente attivo e un punto di ripristino attuale verificato. |
| `complete` | Nessuno | Nessuno. Controlli di compact e un’esecuzione attiva. |
| `status` | Nessuno | Nessuno. Richiede stato già attivato. |

I file assegnati restano percorsi esatti relativi al progetto, protetti contro alias Windows, uscite dal perimetro e collegamenti simbolici. Ruoli e livelli seguono il catalogo e i controlli di disponibilità esistenti. Una lista di scrittura vuota serve a ruoli di revisione e coordinamento in sola lettura. Le attività sconosciute vengono rifiutate. Non si creano attività implicitamente. `accepted: false` rifiuta il lavoro di chi ha smesso di scrivere. `accepted: true` lo accetta solo dopo i controlli di revisione applicabili.

<a name="raw-usage-lifecycle"></a>

## Ciclo di vita dei dati grezzi di utilizzo

In N06, `context_pin` è esattamente `{path, sha256, manifest_path, manifest_sha256}`, con percorsi relativi al progetto non vuoti e SHA-256 in minuscolo. Il manifest deve essere un file di input immutabile già assegnato all’attività. `prepare` verifica una nuova vista version1 rispetto allo stato corrente e al manifest fissato separatamente. Un messaggio preparato con il riepilogo resta vincolato al protocollo. `dispatch-start` accetta inoltre `current_context` e richiede un nuovo riepilogo sullo stato successivo a prepare con il manifest originale. Tutti i controlli nativi restano attivi. Vedi il [contesto](context.md) per generazione delle viste, blocchi dovuti ai riscontri, controllo del produttore e limiti.

In N07, `status` mantiene la mappa originale degli stati delle attività e aggiunge `task_observations: {contract_version: 1, tasks: ...}`. Esecuzione nativa e accettazione del risultato sono distinte. Un’attività con esecuzione done resta in attesa finché non passa i controlli di finish. La conclusione indipendente registra il link facoltativo esistente `review` all’attività di revisione e richiede identità di sessione native distinte e non vuote. Devono coincidere anche gli hash dei risultati in revisione assegnati e registrati. Schema2 non cambia. Una revisione non può dipendere direttamente o indirettamente dall’accettazione del proprio produttore fissato. Le accettazioni indipendenti storiche senza identità complete di produttore o revisore o senza riferimenti fissati sono esposte come `historical_unrecorded`. Non possono soddisfare una nuova dipendenza di accettazione in prepare, record o dispatch-start. Riepilogo e programma ausiliario applicano lo stesso criterio di indipendenza. Lo storico non viene migrato automaticamente.

Tipi e intervalli malformati generano un errore e lasciano invariati i byte dello stato. Sono esempi `usedPercent: "10"`, conteggi e orari booleani, percentuale 101, tempo di azzeramento negativo, finestra in forma di elenco o metadati non finiti. `rateLimits`, finestre o valori mancanti o null, una misura ben tipizzata priva della finestra da 300 o da 10080 minuti oppure un’osservazione altrimenti valida ma vecchia o futura significano utilizzo indisponibile. Tale comando `usage` valido elimina apposta la vecchia misura e incrementa `usage_errors`. Alla prima occorrenza restituisce `RETRY_USAGE_ONCE`. Alla seconda restituisce `SAVE_AND_STOP` e passa allo stato di arresto in corso, salvo se è già arrestato definitivamente o completato. Non riattiva in silenzio un’esecuzione arrestata. Sono valide percentuali 0 e 100 e frazioni finite. 100 attiva la procedura per la soglia. Un dato malformato non incrementa il contatore delle misure sconosciute. Restano i metadati grezzi, senza sostituire le finestre valide richieste.

<a name="supported-pinned-entry"></a>

## Punto di ingresso supportato e fissato

Tutte le operazioni usano `scripts/candidate.py --manifest <manifest> --manifest-sha256 <accepted-pin> --version 1.1`. Le modalità di esecuzione richiedono anche `--project <absolute-project>` e `--run-id <fresh-run>`.

| Modalità | Input | Comportamento |
| --- | --- | --- |
| `workflow validate` | Nessun input | Verifica pacchetto accettato e regole dei ruoli e associa uno spazio inattivo |
| `workflow <typed-command>` | `--input <json-file>` o standard input | Esegue i comandi tipizzati indicati sopra |
| `coordination` | `--input <json-file>` | Esegue un lotto limitato senza modello e senza invio nativo automatico |
| `watcher` | `start`, `renew`, `stop`, `status`, `once` | Usa il monitor interno fissato e lo stato di competenza dell’esecuzione |
| `install` | `--destination .../agent-workflow-v1-1` esplicita, `--update` facoltativo | Verifica il contenuto e le condizioni di appartenenza dell’installazione |

L’invocazione diretta della CLI di `workflow.py` viene rifiutata. Non ha punti di ingresso per installare, esportare o generare i ruoli nativi. Costruisci i file di distribuzione con `tooling/build-candidate.py`. Per installare in Codex usa solo l’installatore del candidato fissato. Convertire vecchi schemi non è un comando del candidato e non deve mai riattivare una vecchia installazione ritirata.

<a name="input-and-state-boundary"></a>

## Confine tra input e stato

In una riproduzione temporanea, i comandi con booleani nel formato corretto mantengono le proprie decisioni. I valori facoltativi possono usare false come scelta sicura quando assenti. I tipi malformati si rifiutano anche se interpretabili come veri. Il parser rigoroso tutela sintassi JSON e finitezza dei valori. Si rifiutano autorizzazioni in forma di stringa e stati precedenti. I test usano solo progetti temporanei.

<a name="typed-assignment-and-source-chronology-a04--gi01"></a>

## Assegnazione tipizzata e cronologia della fonte (A04 / GI01)

N02 `bind-active-goal` associa l’obiettivo nativo corrente dopo un’inizializzazione lunga senza inventare la fonte umana o alterarne l’orario. Accetta solo stato mancante o esattamente `{schema_version: 2, status: inactive, tasks: {}}`, con le stringhe tra virgolette nel JSON effettivo. Un programma ausiliario migrato, completato, fermato, arrestato definitivamente, già usato o altrimenti non intatto non si può associare. L’osservazione iniziale è un vero riscontro `get_goal` da 0 a 60 secondi. ID di obiettivo e chat, testo dell’obiettivo, ora originale di creazione e ora di osservazione devono corrispondere all’associazione. La creazione nativa può essere molto vecchia, ma non successiva all’osservazione. Obiettivo del comando e ambito dell’obiettivo coincidono. Un ambito di rapporto o più ampio viene rifiutato. Interpretazione e verifica di procedura, distinte e non vuote, documentano il controllo del contesto corrente e delle restrizioni più recenti effettuato dall’agente principale. Il programma non può autenticarlo.

Lo stato conserva `native_binding` come base attuale. `native_binding_events` e `native_binding_ids` sono distinti dai registri di autorizzazione umana. Ogni evento contiene `binding`, `interpretation`, `procedural_check` e l’`app_goal_observation` originale. L’identificativo di associazione resta riservato contro il riuso negli eventi umani. Gli ID di esecuzione e ambito restano riservati assieme allo storico. Per lavorare su questa base servono osservazioni recenti, con testo e creazione dell’obiettivo corrispondenti. L’autorizzazione umana conserva lo schema esatto della fonte e la regola dei 300 secondi dalla fonte originale. Una ripresa umana sostituisce l’attuale base nativa ma conserva il suo storico distinto, come fa un’attivazione umana successiva. Record o riserve nativi malformati, mancanti, duplicati o contraddittori comportano rifiuto anche dopo una continuazione umana. Arresto, istantanea e arresto definitivo si applicano ancora. Una nuova osservazione attiva dell’obiettivo non riavvia automaticamente il programma arrestato.

Schema 2 rifiuta il campo obsoleto `inputs`. Il controllo ripetuto può conservare esplicitamente le stringhe storiche in `context_refs`. Nessuna stringa di contesto è interpretata come percorso o riferimento ad attività. `file_inputs` è un elenco di oggetti esattamente `{path, revision}`. L’assegnazione normalizza ogni percorso di file relativo al progetto, rifiuta duplicati che differiscono solo per maiuscole, uscite dal perimetro, percorsi riservati e collegamenti, e fissa `sha256` e `present`. null e false indicano assenza. I file in input immutabili non possono coincidere con percorsi assegnati in scrittura alla stessa attività. I file scrivibili conservano hash di base distinti, precedenti all’invio. Ogni attività ha una revisione esplicita e un hash di assegnazione che copre tutte le sue condizioni. Sostituire input, revisione o hash e ricalcolare in silenzio non permette di accettare lavoro precedente.

`review_targets` è un elenco di oggetti esattamente `{producer_id, revision}`, ammesso solo ai revisori. Ogni produttore deve esistere a quella revisione e avere una sessione osservata di recente come idle, done o stopped durante assegnazione e revisione. L’assegnazione fissa gli hash esatti e la presenza dei file prodotti. I riferimenti di contesto non conferiscono autorizzazione di revisione. Gli argomenti preparati includono tutti i riferimenti tipizzati. Input dei file, condizioni e risultati dei produttori vengono ricontrollati in prepare, record, finish del revisore e accettazione dell’autore. Si ricontrollano anche le dipendenze accettate. Per A05, gli stati accepted e running mantengono la propria logica. Durante la revisione indipendente, un autore può restare running pur risultando fermo in scrittura.

Se le condizioni di assegnazione non sono più valide, il programma genera `INPUT_INVALIDATED` e una ricevuta `invalidation`. Contiene task_id, revisione di assegnazione, motivo, percorso esatto cambiato se applicabile, hash o revisione attesi ed effettivi, at e affected_task_ids ordinati. Le attività che usano lo stesso percorso entrano nell’invalidazione solo se il proprio hash, la presenza attesa del file o le condizioni d’incarico non corrispondono all’osservazione. Una revisione più recente fissata ai byte o all’assenza attuali rimane valida se viene invalidato un riferimento più vecchio. L’invalidazione segue quindi i veri legami di dipendenza e revisione. Stringhe di contesto e uguaglianza dei percorsi da sole non creano legami. Gli stati operativi non cambiano. `contract_invalid: true` e la ricevuta persistente impediscono di accettare un vecchio risultato o revisione. Ripristinare i byte precedenti non riattiva il vecchio autore o la revisione già accettata. Ripetere il controllo mantiene ricevuta e byte dello stato.

L’osservazione di input e output di revisione o dipendenza rifiuta collegamenti simbolici, junction, hard link, sostituzioni con directory e uscite dal perimetro prima dell’hash. Un output non sicuro invalida produttore, revisioni e attività dipendenti interessate prima dell’errore, in assegnazione della revisione, prepare, record, finish e accettazione dell’autore. La ricevuta conserva l’hash atteso o null, il percorso relativo esatto e una stringa `actual` che inizia con `unsafe: ` e include il motivo di osservazione previsto. Se il file manca, registra null. Errori I/O estranei, come EIO, risalgono senza essere scambiati per modifica delle condizioni. Si controlla il percorso osservato. Non si realizza un’istantanea atomica del file system rispetto a sostituzioni esterne concorrenti.

Le modifiche parziali estranee vengono scartate. In chiamata diretta, `apply` conserva soltanto le invalidazioni nello stato fornito. execute le salva atomicamente prima di sollevare l’errore. Non c’è un comando per azzerare o ricalcolare gli hash. Controlla la ricevuta e assegna esplicitamente un nuovo ID e una nuova revisione con riferimenti aggiornati. I rifiuti per altri motivi mantengono i byte come prima.

GI01 richiede source_created_at e original_turn_id sia nell’autorizzazione sia nella copia esatta dei campi di fonte source_message. observed_at e observed_turn_id descrivono separatamente l’osservazione. La fonte deve precedere l’osservazione e avere al massimo 300 secondi al momento dell’accettazione. Per continuare, la creazione della fonte deve essere successiva ad arresto, arresto definitivo o completamento. Non basta un orario di osservazione successivo. L’orario dell’ultima fonte accettata sopravvive a ripresa e nuova attivazione. La creazione non può retrocedere e orari uguali richiedono lo stesso turno originale. Gli ID di evento e di fonte originale restano riservati contro il riuso tra tipi, orari di osservazione, esecuzioni, chat e continuazioni. Gli eventi osservati devono corrispondere a obiettivo, esecuzione e chat correnti. Un arresto esplicito deve corrispondere anche all’ambito dell’obiettivo. Intenti futuri, politiche e rapporti possono essere registrati senza autorizzare il lavoro o la ripresa nell’obiettivo corrente. L’arresto per soglia è un arresto meccanico del programma ausiliario e non modifica l’osservazione dell’obiettivo dell’app né crea un permesso utente di pausa.

I campi sono riscontri forniti dal chiamante e non autenticano il fornitore. Il programma non intercetta le chiamate dirette agli strumenti nativi dell’obiettivo e non chiama mai update_goal. Il test di regressione controllato con arresto storico, continuazione successiva e riuso del contesto nello stesso turno verifica il rifiuto del programma e l’osservazione attiva dell’obiettivo invariata. N07 aggiunge la vista versionata di esecuzione e accettazione descritta sopra. La misurazione di contesto e cache con un nuovo sottoagente nativo e l’integrazione della pausa A06/GI02 richiedono ancora riscontri separati.
