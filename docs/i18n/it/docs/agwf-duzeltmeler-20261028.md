<a name="agwf-maintenance-on-28-october-2026"></a>

# Manutenzione di AGWF del 28 ottobre 2026

> Resoconto storico della manutenzione. Le osservazioni e le autorizzazioni qui riportate riguardano quell’intervento. Il [README](../README.md) descrive il funzionamento attuale.

Ciascuna delle sei skill di ruolo ha ricevuto il proprio `agents/openai.yaml` con `allow_implicit_invocation: false`. Le descrizioni dei ruoli valgono per gli incarichi preparati da AGWF. Un ruolo si ferma se manca l’incarico o i riscontri sul modello e sul livello di ragionamento effettivi. Le skill di ruolo non avviano il flusso o il monitor. Dalla skill principale è stata rimossa una formulazione secondo cui la selezione di un progetto precedente restava valida.

Sorgente e installazione sono stati allineati sulle letture condivise, sulla sostituzione che preserva i permessi e sui tentativi limitati per i file di stato Windows. `candidate.py` usa lo stesso lettore con condivisione per i file di stato che cambiano. Se un lettore esterno mantiene un blocco, il vecchio file resta al suo posto. Un blocco persistente resta un errore.

Gli strumenti di contesto e utilizzo sono passati nella cartella `tools/` inclusa nel pacchetto, affinché l’installazione dell’utente non dipenda dal checkout del progetto. In questo controllo storico, sorgente, pacchetto e copia installata corrispondevano per 41 file. La vecchia copia dello strumento di contesto nella radice è stata rimossa e i percorsi dei test sono stati aggiornati.

Il resoconto riporta 63 test del pacchetto e degli strumenti e nove test di accesso ai file su Windows. I nove test Windows sono stati eseguiti anche con `-O` e usando i file installati. Sono state controllate sette definizioni di skill e gli hash dei file. Il test del pacchetto relativo alle sei regole mancanti dei ruoli non è passato prima della correzione ed è passato dopo.

AGWF era disattivato durante la manutenzione. Le impostazioni delle sette skill sono rimaste disattivate. I test hanno verificato l’accesso del motore a file temporanei. La convivenza con Remote in esecuzione non è stata provata.

A quel punto i rischi ancora aperti comprendevano l’avvio di un secondo server Codex a ogni misura, la scelta di un’altra CLI da PATH, la perdita dell’output di errore del server figlio e i controlli completi ripetuti del pacchetto. Il primo passaggio non ha accertato la causa del problema di Remote. Il [seguito della manutenzione](agwf-tum-duzeltmeler-20261028.md) descrive le modifiche successive. Il monitor attuale non avvia più un secondo server Codex.
