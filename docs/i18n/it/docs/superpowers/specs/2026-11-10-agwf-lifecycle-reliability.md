<a name="lifecycle-reliability-specification"></a>

# Specifica di affidabilità del ciclo di vita

> Specifica storica del 10 novembre 2026. I riscontri e le autorizzazioni riportati riguardano quell’intervento. Il comportamento attuale è descritto nella [guida operativa](../../../skills/agent-workflow-v1-1/references/operations.md).

<a name="intended-result"></a>

## Risultato atteso

Rendere affidabili prima attivazione, aggiornamento delle misure, ripresa e chiusura ordinaria. Preparare il test prima delle esecuzioni previste. Osservare e riportare separatamente avvio, ripresa da un flusso effettivamente arrestato e tempi.

<a name="evidence-available-at-the-time"></a>

## Riscontri disponibili all’epoca

- Il rapporto temporale privato originario registrava una misurazione scaduta prima dell’attivazione. Entrambe le prove a dieci secondi non sono riuscite, con 98,74 e 45,05 secondi. Una nuova epoca e una prima associazione nella seconda prova non dimostravano la ripresa di un programma ausiliario precedentemente arrestato.
- Un altro rapporto privato, sul flusso ordinario, registrava avvio riuscito, seconda misurazione reale dopo 43,423 secondi, punto di ripristino verificato, chiusura ordinaria e uscita del processo. L’esecuzione non ha verificato un blocco SAVE.
- All’epoca `scripts/candidate.py` poteva aprire una nuova epoca tecnica e ripristinare i controlli dopo un `resume` non riuscito. Il primo `activate` o `bind-active-goal` non aveva lo stesso registro dei tentativi. L’implementazione richiedeva riscontri di regressione distinti.
- Una manutenzione precedente aveva corretto conflitti tra autorizzazioni e istruzioni, oltre a quattro anomalie relative alle funzioni di test e alle attese. Questi comportamenti dovevano restare coperti dai test di regressione.

<a name="requirements"></a>

## Requisiti

1. Riprovare un primo avvio con l’autorizzazione esistente solo dopo un `usage_receipt_expired` registrato nello stesso ambito di obiettivo ed esecuzione, con una nuova misura reale, un’osservazione effettiva dell’obiettivo attivo e assenza verificata del processo. Non dovrebbe servire una nuova citazione dell’utente, una nuova identità di esecuzione o la riscrittura dei dati d’uso.
2. Rifiutare quota, riserva anticipata, arresto dell’utente, obiettivo inattivo, appartenenza del processo incerta, pacchetto o riscontri modificati e obiettivo diverso. Python controlla formato e coerenza. Non autentica in modo indipendente la persona o lo strumento indicati dal chiamante.
3. Rendere visibili i tentativi falliti. Un’esecuzione inizializzata solo in parte non deve apparire attiva. Conservare successivi riscontri SAVE reali.
4. Preparare pacchetto e test prima della prima misura. Conservare l’orario effettivo di osservazione. Riscrivere il file non rende nuova una misura precedente. Mantenere `ingest` separato dal rinnovo del lease.
5. In condizioni normali lasciare che `CHECKPOINT_SOON` e `SAVE_AND_STOP` dipendano da decisioni reali. Non iniettare un arresto.
6. Dimostrare Continue nell’interfaccia nativa e la ripresa del flusso dopo aver osservato che era davvero attivo e poi arrestato. Tenere questa verifica distinta dalla prima associazione.
7. Eseguire la diagnosi temporale solo su richiesta. Preparare lo strumento prima del vero evento nativo. Un’osservazione tardiva o una preparazione oltre la scadenza non possono passare. La tolleranza proposta era di 10 ± 1 secondi per chiedere SAVE, con registrazione del tempo reale.
8. L’agente principale continua a chiamare gli strumenti nativi per l’obiettivo sull’host. Non aggiungere al processo in background accessi host non documentati, un trasporto privato Codex o lettura di credenziali. Una misura precisa dal clic nell’interfaccia richiederebbe una vera fonte di eventi nativi.
9. Verificare separatamente punto di ripristino, uscita del processo, completamento del programma ausiliario e completamento dell’obiettivo nativo. Mettere in pausa soltanto con autorizzazione dell’utente e usare `blocked` soltanto secondo i criteri dello strumento nativo.
10. Mantenere Python 3.11 o successivo e la libreria standard. Un errore di condivisione Windows non prova né l’uscita del processo né il successo. Preservare le modifiche locali esistenti e i vecchi record di esecuzione senza riscriverne le associazioni.

<a name="original-authorization"></a>

## Autorizzazione originale

Il documento è nato come attività di pianificazione. In seguito l’utente ha autorizzato, per quell’intervento, implementazione, commit e push del sorgente v1.1 e aggiornamento della documentazione. Una release GitHub e un tag erano fuori ambito. L’arresto temporizzato effettivo restava una diagnosi da selezionare separatamente. Le vecchie associazioni di esecuzione sono state preservate.
