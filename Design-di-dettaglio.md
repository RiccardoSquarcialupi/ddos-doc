In questo capitolo verrà analizzata in maniera dettagliata la struttura del framework, andando a descrivere i singoli componenti e le relazioni fra di essi.

## Package: Device

### Attuatore

![Actuator UML](https://i.imgur.com/Ybv1jdD.png)

La classe `Actuator` è un'implementazione del *trait* `Device`. 


## Package: Storage

![TuSoW packages map](https://raw.githubusercontent.com/Filocava99/TuSoW/master/project-map.svg)

Il modulo relativo allo storage dei dati è, ad oggi, basato interamente su TuSoW.  
TuSoW è una piattaforma per la gestione di spazi delle tuple di LINDA in un ambiente multi-thread, multi-processo, distribuito o multi-agente. È stato scelto poichè è possibile utilizzare come *template* per la ricerca e scrittura di tuple, sia template testuali (i.e. regular expression) sia template logici (i.e. Prolog).

Nello specifico, è stato utilizzato un fork di TuSoW realizzato da Cavallari per il progetto di sistemi distruibuiti il quale introduceva il supporto per gRPC; in questo modo, è stato possibile riutilizzare le definizioni in `protobuf` dei messaggi e dei servizi anche su Akka gRPC.  

### Messaggi
I messaggi, una volta compilati in Scala dal compilatore protobuf di Akka gRPC, sono i seguenti:

![](https://i.imgur.com/b4GtP5S.png)

Rapida spiegazione di alcune parole chiave usate nell'UML:

-   `repeated` indica che il campo può essere valorizzato con uno o infiniti valori
    
-   `oneof` indica che può essere valorizzato solo uno dei campi definiti con _oneof_
    
-   l'etichetta `interface` è usata per rappresentare messaggi che includo messaggi figli
    
-   l'etichetta `ENUM` è usata per identificare messaggi di tipo Enum

##### TupleSpaceID

Il messaggio _TupleSpaceID_ contiene sia l'ID di un _tuple space_ sia il suo tipo (_logic_ o _textual_).

##### TupleSpaceType

Questo messaggio è banale _enum_ usato per specificare la tipologia del *tuple space*.

##### IOResponse / IOResponseList

Il messaggio _IOResponse_ è il risultato delle operazioni di input e restituisce un feedback sull'esito dell'operazione usando un valore booleano ed una stringa con una descrizione dettagliata.
L'*IOResponseList* è una lista di * IOResponse*; viene create usanda la keyword `repeated`.

##### Tuple / TupleList

Il messaggio *Tuple* astrae il concetto di tupla, definendo due attributi, uno per il template usato per eseguire la query e l'altro contenente il valore della tupla. Anche in questo caso esiste la variante lista chiamata *TupleList*.

##### Template / TemplatesList

Il messaggio _Template_ definisce due messaggi figli:

-   _Template.Logic_: rappresenta un template da usare in un *tuple space* logico (i.e. Prolog)
    
-   _Template.Textual_: rappresenta un template da usare in un *tuple space* testuale (i.e. Regex)

Le varianti per valori multipli sono i due messaggi figli di *TemplateList*:
* _TemplatesList.LogicTemplatesList_
* _TemplatesList.TextualTemplatesList_

##### ReadOrTakeRequest / ReadOrTakeAllRequest

*ReadOrTakeRequest* è utilizzato per recuperare una tupla da un *tuple space* utilizzando un template per eseguire la ricerca. È possibile utilizzare il messaggio *ReadOrTakeAllRequest* per utilizzare una lista di template e recuperare più tuple in una sola operazione.

##### WriteRequest / WriteAllRequest

Il messagio _WriteRequest_ contiene una tupla e l'ID dello spazio delle tuple in cui inserirla. Si possono inserire più tuple con una sola operazione utilizzando il messaggio *WriteAllRequest*.


### Servizi
L'unico servizio implementato è il TusowService, il quale mette a disposizione diverse *remote procedures*.

![](https://i.imgur.com/k6oDVtS.png)

Nello specifico le procedure sono:

-   `validateTupleSpace`: permette di verificare l'esistenza di un *tuple space*;
    
-   `createTupleSpace`: crea un nuovo *tuple space*;
    
-   `write`: aggiunge una tupla ad uno spazio delle tuple;
    
-   `read`: recupera una tupla dallo spazio delle tuple che combacia con il template utilizzato;
    
-   `take`: recupera e poi rimuove una tupla dallo spazio delle tuple che combacia anchessa con il template utilizzato;
    
-   `writeAll`: aggiunge tutte le tuple fornite nella lista;
    
-   `readAll`: recupera tutte le tuple che combaciano con la lista di template fornita;
    
-   `takeAll`: recupera ed elimina tutte le tuple che combaciano con la lista di template fornita;
    
-   `<write/read/take>AllAsStream`: variante _streamed_ di _writeAll_, _readAll_, _takeAll_; _streamed_ significa che viene aperto uno *stream* di dati nel quale vengono inviati i risultati delle operazioni uno per volta.

### Integrazione con Akka gRPC

Lo sviluppo ha richiesto di scrivere l'implementazione delle *remote procedure* in Scala in modo che potessero essere incapsulate all'interno di un attore di Akka.