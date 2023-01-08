### Requisiti di business

L'obiettivo è quello di realizzare una libreria che faciliti l'utilizzo di qualunque tipologia di dispositivo IoT, nello specifico di sensori ed attuatori. La libreria metterà a disposizione un insieme di oggetti e di funzionalità che permetteranno all'utente di creare e gestire i propri dispositivi. In particolare, si darà la possibilità all'utente di:

* Creare sensori ed attuatori di ogni tipologia;
* Creare gruppi virtuali di sensori;
* Effettuare il *deploy* dei dispositivi in un ambiente distribuito;
* Gestione semplificata della persistenza dei dati.

Dopo che l'utente avrà definito i propri dispositivi e le proprie zone, potrà anche monitorare il funzionamento complessivo mediante un'interfaccia grafica dedicata.

### Requisiti utente

Di seguito sono riportati i requisiti visti nell'ottica di cosa può fare l'utente:

* l'utente potrà creare, modificare od estendere nuove tipologie di sensori a partire dalle implementazioni già fornite dal framework;
* analogamento, l'utente potrà creare, modificare od estendere nuove tipologie di attuatori basandosi sulle implementazioni esistenti;
* l'utente potrà definire la logica di funzionamento di un attuatore da lui creato;
* l'utente potrà definire gruppi di dispositivi e la loro logica di funzionamento;
* l'utente potrà salvare i dati rilevati e/o prodotti dai dispositivi;
* l'utente potrà visualizzare il comportamento dei propri dispositivi mediante un'interfaccia grafica simulativa.

### Requisiti funzionali

Di seguito sono riportati i requisiti individuati durante lo studio del dominio e le regole scelte per la sua rappresentazione.

#### Dispositivi
* I dispositivi si possono creare non temporizzati o temporizzati (usando il modulo `Timer`) e/o con la funzione di invio di ogni nuovo valore rilevato in broadcast a tutti gli altri dispositivi (usando il modulo `Public`);
* I dispositivi inviano in broadcast ad alcuni dispositivi il nuovo valore rilevato; 
* I dispositivi temporizzati ad ogni `Tick` inviano in broadcast ad alcuni dispositivi il nuovo valore rilevato;
* I dispositivi pubblici inviano in broadcast a tutti gli altri dispositivi il nuovo valore rilevato;
* I dispositivi temporizzati pubblici ad ogni `Tick` inviano in broadcast a tutti gli altri dispositivi il nuovo valore rilevato.

#### Sensori
* I sensori sono dispositivi in grado di effettuare una pre-elaborazione del dato - prima di inviarlo ad altri sensori e/o di memorizzarlo - e di aggiornare il proprio stato con l'ultimo valore rilevato;
* I sensori sono dispositivi che si possono creare con i moduli di `Device` (`Timer` e `Public`) e/o con il modulo `Condition`, oppure senza alcun modulo;
* I sensori `Condition` inviano il proprio stato dopo averlo aggiornato se e solo se si verifica una determinata condizione specificata dall'utente;
* Il `BasicSensor` è un sensore concreto che effettua una pre-elaborazione di base del dato;
* Il `ProcessedDataSensor` è un sensore concreto che effettua una qualsiasi pre-elaborazione del dato specificata direttamente dall'utente.

#### Attuatori
* Gli attuatori sono dispositivi che reagiscono ad eventi interni e/o esterni variando il loro stato e propagandone l'avvenimento;
* I possibili stati di un attuatore devono essere definiti con una macchina a stati finiti;
* Gli stati possono avere una logica basilare oppure variare in base ad un timer o ad una condizione applicata ad un evento;
* Poichè gli attuatori sono dispositivi, deve essere possibile utilizzare tutti i moduli definiti per i dispositivi generici.


#### Gruppi
* Un gruppo di dispositivi deve applicare una funzione definita dall'utente agli output prodotti dai membri di quel gruppo, producendo un nuovo risultato aggregato.
* I gruppi possono essere innestati.
* A scelta dell'utente è possibile creare un gruppo che aspetta (o non aspetta) di aver collezionato i valori di tutte le sue sorgenti prima di computare la propria funzione.
* Sono fornite due possibili tipi di funzioni di aggregazione: la riduzione (cioè la fusione dei valori delle sorgenti in un solo valore) e il mapping (cioè la trasformazione di ogni singolo elemento di input secondo la funzione di computazione).
* Dev'essere possibile astrarre la creazione di un gruppo dall'ordine di generazione degli attori che ne costituiscono le sorgenti.
* Dev'essere compatibile con il metodo di deployment adottato dal modulo apposito.

#### Deploy
* La complessità del *deploy* dei dispositivi in ambiente distribuito deve essere mascherata il più possibile;
* la configurazione dell'ambiente deve essere consentita tramite apposite funzioni, fra cui l'aggiunta di nuovi nodi del *cluster* ;
* sarà possibile *deployare* i dispositivi utilizzando un grafo di dipendenze; ogni arco indicherà la relazione publish/subscribe fra i due dispositivi connessi.

#### Storage
* fra i vari dispositivi dovrà essere possibile crearne uno virtuale che riceve i dati da ogni altro dispositivo per poi salvarli sulla tipologia di gestione dei dati scelta;
* l'unica modalità di persistenza dei dati offerta al rilascio del framework sarà TuSoW
* per il corretto funzionamento di TuSoW in ambiente Scala sarà necessario effettuare delle modifiche mirate, volte anche all'integrazione con librerie terze.

### Requisiti non funzionali

Di seguito sono descritti i requisiti non funzionali dell'applicativo:

* ***Usabilità***: facilità nell'utilizzo del framework per realizzare sistemi distribuiti su larga scala; in particolare bisogna ridurre al minimo i requisiti di conoscenza a basso livello delle librerie utilizzate per l'implementazione in modo da appiattire la curva di apprendimento, anche attraverso DSL appositi.
* ***Cross Platform***: sarà possibile eseguire il sistema sui 3 principali sistemi operativi: Linux, Windows, MacOs.

### Requisiti di implementazione

ll sistema sarà sviluppato in Scala 3 ed utilizza le seguenti librerie:

* Akka-Actor-Teskit-Typed, Akka-Serialization-Jackson Akka-Actor-Typed, Akka-Cluster-Typed v2.7.0
* ScalaTest v3.2.14
* TuSoW v0.8.3
* ScalaFX v19

Lo unit testing è effettuato utilizzando ScalaTest e Akka TestKit.

Il codice sorgente è controllato ad ogni compilazione mediante l'utilizzo del linter ScalaFMT.
