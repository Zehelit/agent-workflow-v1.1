<a name="agwf-monitoring-follow-up-on-28-october-2026"></a>

# Seguito della manutenzione del monitor AGWF del 28 ottobre 2026

> Resoconto storico. Descrive le attività successive al [primo intervento](agwf-duzeltmeler-20261028.md) e non conferisce alcuna autorizzazione all’attivazione attuale.

| Riscontro | Modifica e verifica registrate all’epoca |
| --- | --- |
| Le sei skill di ruolo potevano essere selezionate automaticamente | Ciascun ruolo ha ricevuto `allow_implicit_invocation: false` e un perimetro limitato agli incarichi preparati. I test del pacchetto hanno verificato tutte e sette le skill. |
| Sorgente, repository remoto e installazione non coincidevano | Gli strumenti obbligatori sono stati inclusi nel pacchetto. I file installati sono stati confrontati con il manifest e lo ZIP costruiti da un commit sorgente pulito. |
| Lettura e scrittura entravano in conflitto su Windows | I componenti coinvolti hanno adottato letture condivise e nuove modalità di sostituzione. I test Windows su file temporanei hanno coperto blocchi, nuovi tentativi e conservazione del vecchio contenuto. |
| A ogni misurazione si avviava un secondo server Codex | Quel codice è stato rimosso. L’agente principale consulta lo strumento d’uso già disponibile nell’host e Python legge il rilevamento. I test di misurazione bloccano le chiamate di avvio dei processi. |
| PATH poteva scegliere un’altra CLI Codex | La scelta dell’eseguibile è stata rimossa dal percorso di misurazione. |
| L’output di errore veniva scartato | Il server figlio è stato rimosso. Gli errori di misura conservano fino a 20 record con codice diagnostico, classe di errore e fase. Gli errori imprevisti del processo Python conservano il suo stderr locale. |
| Due cicli controllavano nuovamente il pacchetto ogni secondo | Il segnale periodico verifica solo l’identità dell’esecuzione e della sua epoca. Il controllo completo si effettua ai comandi e alle misurazioni. |
| I test usavano un server simulato | I test sono passati al contratto dei rilevamenti d’uso. Il resoconto riferisce anche l’esame di una vera risposta Codex tramite un file temporaneo. |
| La regola del ruolo riportava ancora una soglia errata | Il vecchio testo `%10` è stato allineato alla soglia di arresto effettiva `%20`. Un avvio rifiutato per dati mancanti rimuove il relativo indicatore di attività. |

Il nuovo percorso dati usa l’orario di osservazione originario e l’identità della fonte tramite `--usage-input`. Il monitor lega il rilevamento alla propria esecuzione ed epoca. Rifiuta misure mancanti, future, scadute, anteriori alle precedenti o riferite a un’altra esecuzione. Ripubblicare la stessa misura non può aggiornare l’ora di osservazione. Se manca un nuovo input, il monitor ritenta la convalida e poi registra SAVE_AND_STOP.

L’agente principale raccoglie le nuove osservazioni prima della scadenza. Python non chiama lo strumento dell’host. `ingest` pubblica solo i dati. Attivazione, avvio del processo, autorizzazione, aggiornamento dei riscontri sull’obiettivo e rimozione di un arresto sono operazioni separate.

Il resoconto riporta 111 test mirati nelle modalità standard e `-O`. In quella suite gli avvii reali di AGWF e Codex erano bloccati. Il programma di build e i test degli strumenti in sola lettura potevano usare i sottoprocessi ammessi. I controlli del sorgente, gli hash dell’installazione, le skill disattivate e la riproduzione da sorgente pulito sono stati registrati separatamente.

Il resoconto riporta inoltre il passaggio di un risultato d’uso reale attraverso il percorso previsto senza avviare un altro processo o cambiare l’ora dell’osservazione. Nel pacchetto non sono stati aggiunti dati dell’account o dell’identità. La causa esatta del problema storico di Remote non è stata verificata e la convivenza effettiva con AGWF non faceva parte del test. Vedi la [specifica dei rilevamenti nativi](../skills/agent-workflow-v1-1/references/native-usage.md).
