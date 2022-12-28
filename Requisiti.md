### Requisiti di Business

L'obiettivo è quello di realizzare una libreria che faciliti l'utilizzo di qualsiasi tipo di sensori e di attuatori in un qualsiasi contesto IoT. La libreria metterà a disposizione un insieme di oggetti e di funzionalità che permetteranno all'utente di creare e gestire i propri dispositivi. In particolare, si darà la possibilità all'utente di:

-   Creare sensori ed attuatori di ogni tipologia;
-   Creare gruppi virtuali di sensori
- //TODO

### Requisiti Utente

Di seguito sono riportati i requisiti visti nell'ottica di cosa può fare l'utente.

-   L'utente potrà creare nuove tipologie di sensori "custom" a partire da quelle già presenti o modificare/estendere quelle presenti;
-   L'utente potrà creare nuove tipologie di attuatori "custom" a partire da quelle già presenti o modificare/estendere quelle presenti;
-   //TODO

### Requisiti Funzionali

Di seguito sono riportati i requisiti individuati durante lo studio del dominio e le regole scelte per la sua rappresentazione.

-   //TODO

### Requisiti non Funzionali

Di seguito sono descritti i requisiti non funzionali dell'applicativo:

-   _**Usabilità**_: //TODO
-   _**User Experience**_: //TODO
-   _**Cross Platform**_: Sarà possibile eseguire il sistema sui 3 principali sistemi operativi: Linux, Windows, MacOs.

### Requisiti di implementazione

Di seguito vengono riportati i requisiti relativi all'implementazione del sistema:

-   Il sistema sarà sviluppato in Scala 3.* ed utilizza le seguenti librerie:
			--Akka - Akka-Actor-Typed - Akka-Cluster-Typed v2.7.0
			--Monix v3.4.1
			--JFreeChart v1.5.3 
			--ScalaTest v3.2.14
			--TuProlog v4.1.1
			--TuSoW v0.8.3
			--ScalaFX v19
-   Il sistema farà riferimento al JDK 18, eventuali librerie esterne utilizzate dovranno supportare almeno tale versione;
-   Il testing del sistema sarà effettuato utilizzando ScalaTest e Akka TestKit utilizzando in questo modo sarà minimizzata la presenza di errori e facilitato l'aggiornamento di eventuali funzionalità;
-   Il codice sorgente sarà verificato mediante l'utilizzo del linter ScalaFMT. (?)