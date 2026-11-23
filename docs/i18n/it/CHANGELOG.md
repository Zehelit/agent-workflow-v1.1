<a name="changelog"></a>

# Cronologia delle modifiche

Questo documento raccoglie la storia delle versioni esistenti. Il commit di origine e la ricevuta di build accettata permettono di identificare il pacchetto esatto.

<a name="11--2026-11-10"></a>

## 1.1 — 10 novembre 2026

<a name="added"></a>

### Aggiunte

- Traccia persistente del primo tentativo con activate o bind-active-goal. Per riprovare in sicurezza si usa lo SHA-256 della ricevuta di errore, conservato fuori dal pacchetto.
- Un blocco di scrittura appartenente all’esecuzione per serializzare le operazioni del ciclo di vita del candidato.
- tooling/run-lifecycle-check.py per preparazione, avvio, aggiornamento delle osservazioni, ripresa dopo arresto e chiusura ordinaria.
- Modalità timed-test esplicita, con orari separati per SAVE, punto di ripristino e arresto e una classificazione degli eventi tardivi. Lo strumento non accerta la pausa nativa.
- Orario effettivo della misurazione, scadenza, intervallo e suggerimento per un aggiornamento anticipato nei riepiloghi di stato.
- Scopo del progetto, diagrammi di architettura, flusso di lavoro e orchestrazione, istruzioni di installazione manuale e tramite agente e guida al ciclo di vita.

<a name="fixed"></a>

### Correzioni

- Ripresa controllata con lo stesso obiettivo attivo e autorizzato dopo la scadenza del rilevamento d’uso. Gli arresti per quota esaurita o per richiesta dell’utente seguono procedure proprie.
- Il nuovo tentativo dopo un primo avvio non riuscito conserva ambito, riscontri SAVE precedenti, associazione al pacchetto e verifica di assenza del processo.
- L’eccezione temporanea alla finestra settimanale si applica allo stesso modo nel motore e nel monitor, senza riscrivere i valori osservati.
- I tentativi limitati gestiscono determinati errori temporanei di condivisione e lettura dei blocchi su Windows. Gli errori permanenti o non classificati restano visibili.
- Chiariti aggiornamento della misurazione, rinnovo del lease e requisiti temporali dei riscontri dopo l’arresto.
- Aggiornate vecchie firme degli strumenti di test e le attese relative a processi e file.

<a name="compatibility-and-evidence"></a>

### Compatibilità e riscontri

- Nome d’invocazione, directory d’installazione e percorso dei dati di esecuzione passano insieme alla 1.1. I nuovi nomi sono `$agent-workflow-v1-1`, `agent-workflow-v1-1/` e `.agent-workflow-v1-1/runs/`.
- I nuovi manifest rifiutano la vecchia identità tecnica. Le esecuzioni precedenti restano come riscontri storici e non si possono associare automaticamente al nuovo pacchetto. I vecchi percorsi restano riservati per verificare l’appartenenza e ripristinare i dati.
- Un aggiornamento non riscrive l’associazione immutabile di un’esecuzione. Usa una nuova esecuzione dopo aver approvato il nuovo pacchetto.
- Sono richiesti Python 3.11 o successivo e la sola libreria standard. Il monitor non accede autonomamente allo strumento nativo di utilizzo e non apre un secondo server Codex.
- I test isolati usano osservazioni sintetiche dell’host e veri processi Python. Remote, Continue e il tempo misurato dall’interfaccia attiva richiedono riscontri distinti.
- La versione è stata registrata con commit del codice sorgente. Una release GitHub e un tag erano fuori dal perimetro di pubblicazione registrato.

<a name="10--previous-source-version"></a>

## 1.0 — versione precedente del codice

- Punto di ingresso del candidato fissato tramite hash del manifest, stato tipizzato, documenti di ruolo, controlli sulle responsabilità dei file e sulla capacità.
- Monitor integrato senza modello, punti di ripristino e ripristino, registro di coordinamento persistente e pacchetto deterministico.
- Lo storico non stabilisce se esista una release GitHub o un tag corrispondente.
