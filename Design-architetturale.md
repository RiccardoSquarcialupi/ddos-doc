In seguito all'analisi dei requisiti del framework, si è voluto sviluppare il progetto adottando una struttura estremamente modulare, in modo da rendere opzionale l'uso di determinate funzionalità in base alle necessità dello sviluppatore/utilizzatore.

![Modules UML](https://i.imgur.com/Q6UsOYE.png)

Le dipendenze fra moduli sono evidenziate dalle frecce direzionali; l'unico modulo indipendente è il modulo dei dispositivi (**Devices**), fornendo tutte le strutture dati da cui dipendono gli altri moduli.  
Il modulo **Deployment** fornisce funzioni per il deploy all'interno di cluster dei vari dispositivi, nascondendo completamente all'utente la complessa logica implementativa sottostante.  
Il modulo **Grouping** contiene classi e funzioni per gestire le aggregazioni di dispositivi.  
Il modulo **Storage**, invece, offre delle API per permettere la persistenza dei dati.  
Per concludere, il modulo **GUI**, strutturato secondo il pattern MVC, permette di visualizzare graficamente lo status e i dispostivi all'interno del network.  