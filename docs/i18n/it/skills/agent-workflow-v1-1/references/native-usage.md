<a name="native-usage-receipts"></a>

# Rilevamenti nativi di utilizzo

Usa questa procedura solo in un’esecuzione AGWF attiva e autorizzata esplicitamente. Un’invocazione esplicita attuale della skill richiede l’avvio nel perimetro autorizzato. Se è vietato creare un obiettivo nativo, usa il monitoraggio manuale descritto nella [guida operativa](operations.md). Aggiornare il pacchetto, parlare della skill o pubblicare dati d’uso non riattiva un flusso disabilitato.

L’agente principale chiama `mcp__codex_app__get_usage_limits` dalla connessione Codex esistente. Python non apre un secondo server applicativo, non seleziona un’altra CLI, non legge file di autenticazione, non si collega a trasporti privati e non chiama modelli. Il processo in background controlla le ricevute fornite e la loro validità temporale. Nei passaggi chiave è comunque l’agente principale a chiamare lo strumento dell’host.

Crea un JSON di input privato da un risultato reale dello strumento.

```json
{
  "source": "mcp__codex_app__get_usage_limits",
  "sourceId": "actual-tool-call-receipt-id",
  "observedAt": 1800000000,
  "result": {
    "rateLimits": {
      "limitId": "codex",
      "primary": {"usedPercent": 25, "windowDurationMins": 300, "resetsAt": 1800010000},
      "secondary": {"usedPercent": 30, "windowDurationMins": 10080, "resetsAt": 1800600000}
    }
  }
}
```

I numeri dell’esempio sono illustrativi. Conserva l’ora di osservazione effettiva, l’identità della ricevuta e i valori misurati. Non inventarli e non sostituire l’orario di osservazione con quello di pubblicazione del file. Preferisci `rateLimitsByLimitId.codex` se il risultato effettivo ha più raggruppamenti. Se esiste la mappa ma non contiene Codex, il programma rifiuta l’input senza usare un altro raggruppamento in automatico. Sono necessarie entrambe le finestre con durata e orario di azzeramento. Copia nell’input privato solo i campi necessari. I dati di account, profilo e credito non servono. Il file pubblicato `watcher/usage-source.json` conserva soltanto i campi delle finestre approvati, identità e orario della fonte e dell’osservazione, e l’associazione all’esecuzione e alla sua epoca.

`workflow activate --input AUTH.json --usage-input USAGE.json` e `bind-active-goal` verificano l’autorizzazione prima di avviare un processo. Anche `resume` richiede una nuova misura passata con `--usage-input`. Una ricevuta della vecchia epoca non si può riusare. Il monitoraggio manuale con `watcher start --usage-input USAGE.json` segue gli stessi controlli preliminari. Una ricevuta mancante o già scaduta fa rifiutare l’avvio prima che il processo sia creato.

Per trasmettere una misura recente durante il lavoro, usa `watcher ingest --usage-input USAGE.json` oppure l’opzione `--usage-input` di un comando di flusso o coordinamento. `ingest` non attiva il flusso, non avvia processi, non rinnova l’autorizzazione e non rimuove un blocco SAVE. Un nuovo file sorgente attiva una valutazione senza attendere la scadenza precedente. Lo stato può mostrare brevemente la valutazione precedente mentre il processo di competenza acquisisce il dato. Attendi uno stato aggiornato e operativo prima di un invio rilevante.

L’agente principale aggiorna la misura reale prima della scadenza adattiva. L’intervallo massimo è 300 secondi e si accorcia vicino alle soglie o se il consumo aumenta rapidamente. Rileggere un vecchio file non produce una nuova misurazione. Se l’agente principale non ottiene dati aggiornati, il processo ritenta la convalida una volta e poi registra il blocco SAVE_AND_STOP. Un arresto registrato come `usage_receipt_expired` può riprendere con lo stesso obiettivo già autorizzato e ancora attivo tramite `workflow resume` e `usage_recovery: true`. Vedi [ripresa tecnica](operations.md#expired-telemetry-recovery-under-existing-authority). I dati recenti da soli non annullano il blocco. La ripresa verificata non richiede un altro messaggio dell’utente. Quota esaurita e arresti espliciti dell’utente conservano le proprie condizioni. Aggiorna ai passaggi chiave, in tempo per la prossima chiamata. La scadenza non è un’attesa consigliata. Chiamate lunghe e non interrompibili limitano gli aggiornamenti tramite l’host. L’accesso autonomo in background allo strumento nativo non è garantito.

Il programma rifiuta input malformati, esecuzione o epoca errate, osservazioni future o scadute, orari che tornano indietro e ricevute modificate e ripresentate. Ripubblicare una ricevuta identica non produce modifiche. Come nelle altre ricevute native, orario e fonte sono informazioni fornite dal chiamante. La sola struttura JSON non autentica il fornitore. Conserva in privato i riscontri grezzi e non considerare un’etichetta arbitraria come prova indipendente.

Gli errori di misurazione espongono un `diagnosticCode` consentito e la classe dell’errore nella misura e nello stato. `watcher/diagnostics.json` conserva fino a 20 voci strutturate. Il testo delle eccezioni, i dati di autenticazione e i contenuti arbitrari degli strumenti sono esclusi. Gli errori inattesi del processo Python restano nel file locale `worker-stderr.log`. L’output di errore del server figlio non viene più scartato perché quel server non esiste più in questa procedura.

A ogni comando del candidato e a ogni valutazione del processo, anche su nuovi input pubblicati, si controlla l’integrità dell’intero pacchetto. Il segnale periodico verifica solo i dati di esecuzione, epoca e controllo e ne pubblica lo stato. Non attraversa e non calcola di nuovo l’hash di tutto il pacchetto. Una dipendenza modificata viene rifiutata al comando o alla misurazione successiva. Non è detto che venga rilevata con ogni segnale.

I test unitari coprono scrittura e lettura effettive dei file di rilevamento, JSON rigoroso, scadenza, riuso, identità, soglie, diagnostica limitata e assenza di avvio di processi nel percorso di misura. L’accettazione effettiva in Remote richiede riscontri propri. Mantieni il flusso installato disabilitato fino alla riattivazione esplicita dell’utente. La manutenzione non deve avviare processi AGWF effettivi per fingere quel risultato.
