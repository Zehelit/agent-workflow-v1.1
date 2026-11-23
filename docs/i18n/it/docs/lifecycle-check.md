<a name="repeatable-lifecycle-check"></a>

# Verificare il ciclo di vita in modo ripetibile

`tooling/run-lifecycle-check.py` elabora tramite il candidato fissato le osservazioni reali sull’obiettivo e sull’utilizzo e l’inventario dei sottoagenti. L’agente principale le ottiene dall’host. Tieni sessione e input privati, fuori da Git, e assegna la scrittura della sessione a un solo agente.

<a name="prepare-before-measuring"></a>

## Preparare prima di misurare

Costruisci il candidato e conserva l’hash esterno approvato. `prepare` verifica il pacchetto e registra una nuova sessione senza avviare candidato o monitor. Termina la preparazione prima di acquisire la prima misura d’uso effettiva.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py prepare --candidate C:/build/agent-workflow-v1-1/scripts/candidate.py --manifest C:/build/agent-workflow-v1-1/manifest.json --manifest-sha256 PRESERVED_SHA256 --version 1.1 --project C:/project --run-id check-1 --session C:/private/check-1.json
```

`PRESERVED_SHA256` proviene dalla ricevuta di build accettata. Usa un progetto esistente e un nuovo file di sessione. Se il pacchetto cambia, non associare una vecchia sessione a un altro hash.

<a name="host-input-object"></a>

## Oggetto di input dell’host

`run`, `observe`, `resume`, `close` e `timed-test` accettano un oggetto JSON tramite `--input`.

| Campo | Contenuto |
| --- | --- |
| `activation` | Solo al primo `run`. Dati tipizzati completi del candidato per `bind-active-goal`, con `native_binding`, `app_goal_observation` corrispondente, obiettivo, interpretazione, verifica procedurale, modelli e livelli osservati, capienza, inventario reale e ora di osservazione. |
| `app_goal_observation` | Osservazione recente di `get_goal` con identità di obiettivo e chat, testo dell’obiettivo, ora di creazione e stato, oltre a `source_kind: get_goal`, un riferimento di fonte reale e `observed_at`. Includi il vero `updated_at` se disponibile. |
| `usage` | Campi `source`, `sourceId`, `observedAt` e `result` dei [rilevamenti nativi d’uso](../skills/agent-workflow-v1-1/references/native-usage.md). Mantieni solo le finestre richieste. |
| `inventory` | Identità e stati attuali effettivi dei sottoagenti. Usa `[]` solo in loro assenza. |
| `checkpoint_files` | Percorsi relativi al progetto modificati, aggiunti o eliminati da ripristinare, compresi i file dell’agente principale. Il motore aggiunge anche i file assegnati alle attività. |
| `writers_quiescent` | `true` soltanto dopo avere osservato che tutti gli agenti hanno smesso di scrivere. Lo strumento non può produrre questa osservazione. |
| `next_action` | Passo concreto dopo la ripresa o la chiusura ordinaria. |
| `weekly_exception` | Facoltativo al primo `run`. Citazione effettiva dell’utente `exact_quote` e scadenza Unix delimitata `valid_until`. Nessuna eccezione predefinita. |
| `resume` | Dati completi facoltativi per la ripresa con il candidato. Altrimenti lo strumento tenta `usage_recovery: true` solo per la scadenza della misura registrata. Gli arresti per quota e utente conservano requisiti distinti. |

Se l’API nativa non fornisce un identificativo indipendente dell’obiettivo, usa una derivazione coerente come `threadId:createdAt` e indica che è derivata. I formati tipizzati sono descritti nella [guida operativa](../skills/agent-workflow-v1-1/references/operations.md) e nella [specifica dei comandi](../skills/agent-workflow-v1-1/references/commands.md).

<a name="run-refresh-and-close"></a>

## Eseguire, aggiornare e chiudere

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py run --session C:/private/check-1.json --input C:/private/initial-observation.json
python -X utf8 -B tooling/run-lifecycle-check.py observe --session C:/private/check-1.json --input C:/private/new-observation.json
python -X utf8 -B tooling/run-lifecycle-check.py close --session C:/private/check-1.json --input C:/private/final-observation.json
```

Registra l’ora effettiva di osservazione appena la chiamata nativa ritorna. Lo strumento controlla `status.at == usage.observedAt`, identità del processo di competenza, epoca, segnale periodico, misurazione e lease. `observe` rinnova anche il lease. `refresh_by` suggerisce di aggiornare prima e non sposta la scadenza effettiva.

Con `CHECKPOINT_SOON` verifica un punto di ripristino per il lavoro terminato. Mantieni aggiornato `checkpoint_files`. Il normale `close` richiede monitoraggio recente, istantanea verificata, completamento e uscita del processo osservata. Il normale funzionamento non inietta eventi SAVE.

Un primo `run` con `weekly_exception` installa la regola temporanea del monitor e restituisce `NEED_POLICY_OBSERVATION` prima di attivare il flusso. Raccogli una nuova osservazione su obiettivo e utilizzo e ripeti nella stessa sessione. Conserva l’orario originario di ogni rilevamento precedente.

<a name="halt-and-resume-after-expiry"></a>

## Arrestare e riprendere dopo una scadenza

Quando il monitor chiede SAVE, `observe` applica arresto, istantanea e arresto definitivo, poi restituisce `NEED_POST_HALT_OBSERVATION`. Raccogli nuove osservazioni di obiettivo, utilizzo e inventario dopo l’arresto.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py resume --session C:/private/check-1.json --input C:/private/after-halt-observation.json
```

Il motore usa il motivo osservato dal monitor. La ripresa tecnica richiede lo stesso obiettivo attivo e autorizzato, file e punto di ripristino invariati, nuovi riscontri dopo l’arresto e uscita verificata del processo di competenza. Arresti per quota, riserva anticipata o richiesta esplicita dell’utente richiedono la continuazione applicabile. Lo strumento non imposta lo stato nativo su paused, blocked o active.

Se fallisce la prima attivazione, la sessione mantiene l’hash del tentativo di avvio del candidato. Usa lo stesso `run` con nuovi riscontri soltanto quando sussistono le condizioni di nuovo tentativo. Una sessione modificata non vale come prova. I tentativi completati o incerti non ripartono automaticamente.

<a name="run-a-timed-diagnosis-only-when-selected"></a>

## Diagnosi temporale soltanto su richiesta

Prepara un processo funzionante prima dell’evento. Usa `timed-test` solo per la diagnosi esplicitamente richiesta. Il JSON dell’evento include `event_id`, `goal_id`, `thread_id`, `source_id`, `event_at` e `observed_at` derivati da un’osservazione nativa reale.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py timed-test --session C:/private/check-1.json --input C:/private/current-observation.json --event-input C:/private/native-event.json --delay-seconds 10 --tolerance-seconds 1
```

Un evento tardivo produce `INVALID_TIMING`. `SAVE_REQUEST_ON_TIME` indica che la richiesta di SAVE è arrivata entro la finestra prevista. Per la pausa nativa servono riscontri separati dell’host. Lo strumento lascia quindi `native_pause_verified: false` e `native_paused_at: null`. Un controllo effettivo in due passaggi osserva lo stato attivo, poi l’arresto, poi il nuovo stato attivo dopo una continuazione reale autorizzata con lo stesso obiettivo e la stessa esecuzione. Un test con orologio sintetico esercita solo la logica.
