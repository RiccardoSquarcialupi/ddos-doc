
In questo capitolo verrà analizzata in maniera dettagliata la struttura del framework, andando a descrivere i singoli componenti e le relazioni fra di essi.

## Pattern utilizzati

### Pimp my library

Il pattern *pimp my library* è un pattern che permette di aggiungere nuovi metodi ad una classe senza modificarne il codice sorgente; è particolarmente utile quando si utilizzano librerie terze o classi della standard library di Scala e Java. È stato utilizzato, per esempio, per aggiungere due metodi, `getFirst` e `getSecond`, alla classe `Tuple` di Scala, in modo da restituire un `Option` per gestire valori `null`.

```scala
extension [T](t: (T, T))  
    def getFirst: Option[T] = if(t != null && t._1 != null) Some(t._1) else None  
    def getSecond: Option[T] = if(t != null && t._2 != null) Some(t._2) else None
```

### Strategy

Il meccanismo delle higher-order functions permette di utilizzare in modo triviale il pattern strategy; si potrebbe infattire dire che il pattern strategy è già implementato all'interno di Scala.  
Un esempio di utilizzo è nei ConditionalState in cui il comportamento dello stato viene fornito nel costruttore utilizzando per l'appunto una higher-order function.

```scala
type ConditionalFunction[T] = (T, Seq[T]) => Boolean  
  
object ConditionalState {  
    def apply[T](name: String, condFunc: ConditionalFunction[T]): ConditionalState[T] = new ConditionalState[T](name, condFunc)  
}

class ConditionalState[T](name: String, condFunction: ConditionalFunction[T]) extends State[T](name):

[...] //other code
```

### Singleton

Proprio come il pattern *strategy*, anche il pattern *singleton* è supportato direttamente da Scala.
Poichè il pattern singleton prevede l'esistenza di una sola istanza di una data classe, è sufficiente utilizzare gli `object`. Un esempio di singleton è l'oggetto `Deployer` (mostrati solo i metodi pubblici):

```scala
object Deployer:  
  
  def initSeedNodes(): Unit =  
    ActorSystem(Behaviors.empty, "ClusterSystem", setupClusterConfig("2551"))  
    ActorSystem(Behaviors.empty, "ClusterSystem", setupClusterConfig("2552"))  
      
  def addNodes(numberOfNode: Int): Unit =  
    for (i <- 1 to numberOfNode)  
        val as = createActorSystem("ClusterSystem")  
        Thread.sleep(300)  
        orderedActorSystemRefList += ActorSysWithActor(as, 0)  
  
  def deploy[T](devicesGraph: Graph[Device[T]]): Unit =  
    val alreadyDeployed = mutable.Set[Device[T]]()  
    devicesGraph @-> ((k,edges) => {  
      if(!alreadyDeployed.contains(k)) deploy(k)  
      edges.filter(!alreadyDeployed.contains(_)).foreach(deploy(_))  
    })  
    devicesGraph @-> ((k, v) => v.map(it => devicesActorRefMap.get(it.id)).filter(_.isDefined).foreach(device => devicesActorRefMap(k.id).ref ! Subscribe(device.get.ref)))  
  
```

## Package: Device
### Attuatore

![Actuator UML](https://i.imgur.com/NHrWmrB.png)

La classe `Actuator` è un'implementazione del *trait* `Device`. Come tale può venir istanziato con tutti i mixin disponibili per il `trait Device`, ad esempio `Public`.
```scala
new Actuator[String]("actuatorTest", fsm) with Public[String]
```
Un attuatore può, in base al dispositivo fisico che rappresenta e quindi ad eventi esterni, avere stati differenti; questi stati vengono forniti al costruttore tramite una macchina a stati finiti (classe `FSM`): ogni stato della macchina è un'implementazione del trait `State` ed i pesi degli archi sono i messaggi (ovvero gli eventi) che causano un cambiamento di stato. Ad esempio, si prende come ipotesi l'aver fornito una FSM all'attuatore come quella nell'immagine sottostante:

![FSM di esempio](https://i.imgur.com/WJTiX1A.png)

Se l'attuatore si trovasse nello stato **A** e ricevesse un messaggio *GoTo("B")* allora potrebbe spostarsi nello stato **B**, ma se ricevesse un messaggio *GoTo("D")*, poichè esso non è definito nella FSM, il cambio di stato non verrebbe effettuato.  
In questo scenario abbiamo utilizzato degli stati privi di comportamenti, ovvero dei `BasicState`, tuttavia è possibile utilizzare stati il cui cambiamento è legato a condizioni sull'evento (`ConditionalState`) oppure a dei timer periodici (`TimedState`).
Poichè l'attuatore è un `Device`, maschera al suo interno il `Behavior` di un attore tipizzato di *Akka*; lo scambio dei messaggi e il cambiamento di stato costituiscono infatti il `Behavior` restituito dal metodo `getBehavior` che verrà utilizzato per spawnare l'attuatore all'interno del *cluster*.
Per un uniformare i messaggi scambiati fra i vari attori sia è creato il trait `Message` con varie case class che lo implementano, ad esempio `case class Approved() extends Message`; poichè è necessario che gli utilizzatori finali del framework possano inviare qualunque tipo di oggetto, sono stati creati messaggi che possono incapsulare qualunque tipo utilizzando i generics; ne è un esempio il messaggio `case class MessageWithReply[T](message: T, replyTo: ActorRef[Message], args: T*) extends Message`.

### Sensore

![Sensor UML](https://imgur.com/ewmQv6g.png)

Le classi `BasicSensor` e  `ProcessedDataSensor`  sono un'implementazione del *trait* `Device` e mixin della definizione astratta del *trait* `Sensor`. Come tali possono essere istanziati con tutti i mixin disponibili per il *trait* `Device`, ad esempio `Timer` o `Public` oltre che al mixin `Condition` disponibile per il *trait* `Sensor`  . Possono essere quindi creati sensori per ogni possibile combinazione.
```scala
new BasicSensor[Double]("1", List(testProbe.ref)) with Timer(interval)

new ProcessedDataSensor[String, Int]("1", List(testProbeProcessed.ref), x => x.toInt) with Public[Int]

new BasicSensor[String]("1", List.empty) with Condition[String, String](_ contains "test", testProbe.ref) with Public[String] with Timer(interval)
```
Come l'attuatore, il sensore è un `Device`, ed anch'esso maschera al suo interno il `Behavior` di un attore tipizzato di *Akka*.
Per lo scambio di messaggi si utilizza il trait `Message` come già visto nell'attuatore precedentemente, ma vengono inoltre messi a disposizione dei messaggi  `case class Status[T](author: ActorRef[Message], value: T) extends Output[T](author, value)` per rappresentare l'invio dello stato del sensore, cioè il dato che sta misurando.

## Package: Deployment

### Graph

La classe `Graph` è una struttura dati usata per costruire grafi orientati non pesati.  
Il metodo `apply` è stato pensato per semplificare notevolmente la costruzione di un grafo sfruttando la sintassi delle tuple 2D.

```scala
Graph[String](  
   "A" -> "B",  
   "A" -> "C",  
   "B" -> "D",  
   "C" -> "D",  
   "D" -> "A"  
)
```

Queste coppie di tuple vengono poi raggruppate usando come chiave il primo elemento della tupla, inserendo i secondi elementi in una lista. Si ricava quindi una mappa [K, List[K]] che associa ad un nodo tutti i nodi raggiungibili da esso; poichè il grafo è non pesato, utilizzare una struttura dati per gli archi era superfluo.

Fra i metodi messi a disposizione i più utili sono `@->`, il quale si comporta come un for-each per ogny *entry* della mappa, `?` che verifica l'esistenza di un nodo e `?->` che fornisce la lista dei nodi raggiungibili da un altro nodo (lista vuoto in caso non ci siano archi).

### Deployer

Il *singleton* `Deployer` è l'oggetto centrale del package *deployment*. Esso permette di inizializzare un cluster spawnando i due *seed nodes* definiti a livello di configurazione utilizzando il metodo `initSeedNodes()`; in seguito è possibile aggiungere nuovi nodi al cluster utilizzando il metodo `addNodes(amountOfNodesToSpawn: Int)` (i vari nodi riceveranno una porta casuale fra quelle disponibili).  
La scelta di dover inizializzare manualmente i nodi del cluster è stata effettuata poichè si riteneva, in linea coi requisiti, dare il maggior controllo possibile agli utilizzatori del framework pur mascherando la complessità di Akka sottostante.  

Il metodo `deploy[T](devices: Device[T]*)` è invece il metodo che, preso uno o più `Device` (*varargs*), li spawna come figli dei vari nodi del cluster distribuendoli in modo che ogni nodo abbia lo stesso numero di figli (*load balancing*); tale metodo è però privato, poichè per spawnare i dispositivi si utilizza un metodo omonimo ma con signature diversa: `deploy[T](devicesGraph: Graph[Device[T]])`. Usando un grafo di `Device` è possibile sia spawnare i vari dispositivi, sia definire le relazioni di publish/subscribe fra di essi; questo layer di astrazione aggiuntivo rende più facile il deploy su larga scala di molti dispotivi.

## Package: Grouping

Il modulo Grouping mette a disposizione un tipo particolare di device che permette di definire funzioni higher-order di aggregazione tra output di device diversi. Un gruppo è formato da due componenti principali: l'attore che definisce il behavior akka utilizzato per la gestione dei messaggi e un'istanza che implementa la classe astratta Group che ne definisce lo stato.

![](PPS-Grouping.png)

### GroupActor
Il trait `GroupActor` definisce il behavior Akka del gruppo, ovvero come l'attore reagisce ai vari messaggi che può ricevere. L'attore è stato pensato come macchina a stati finiti composta da due stati: `connecting` e `active`. Per cercare di aderire a uno stile di programmazione puramente funzionale questi due stati sono stati implementati come funzioni ricorsive pure che restituiscono un Behavior akka.
Alla creazione dell'attore, l'oggetto scala che implementa `GroupActor` crea una copia dell'istanza di `Group` con cui è stata invocata l'`apply()` e ne utilizza il campo `sources` per definire la lista di `Device` a cui sottoscriversi e da cui aspettarsi il `SubscribeAck`, passando allo stato `connecting()`.
In questo stato l'attore rimane in attesa di aver collezionato il `SubscribeAck` di tutta la lista di sorgenti, mandando nuovamente il messaggio di `Subscribe` dopo un certo intervallo di tempo grazie al messaggio di `Timeout` che l'attore si invia attraverso un timer interno.
Una volta collezionati tutti gli ack, l'attore passa allo stato `active()`, in cui rimane in attesa dei dati delle varie sorgenti.

Tramite l'override di `getTriggerBehavior()`, che restituisce la logica di innesco della computazione, sono stati implementati due possibili comportamenti per questo stato: bloccante e non bloccante.
L'attore bloccante (`BlockingGroup`) attenderà che tutte le sorgenti abbiano mandato almeno una volta il proprio status (eventualmente sovrascrivendo man mano lo status riportato da una sorgente nel caso questa lo mandasse più volte) prima di richiamare la computazione dell'output.
Al contrario, `NonBlockingGroup` effettuerà la computazione a ogni ricezione di un nuovo status, sia da sorgenti che l'avevano già inviato che da quelle da cui lo sta ricevendo per la prima volta.

Essendo il gruppo stesso un Device, esiste il caso in cui più gruppi siano innestati uno dentro l'altro, creando un potenziale conflitto di semantica tra il tipo T di `Device[T]` e il tipo di output del gruppo, che in certi casi potrebbe consistere in una `List[T]`. Per rendere trasparente all'utente la struttura dati usata per collezionare output multipli, e per poter trattare allo stesso modo gruppi, sensori e attuatori, ogni messaggio di tipo `Status[T]` viene inoltrato dall'attore a sé stesso come messaggio di tipo `Statuses[T]`, simile per concezione a uno `Status[List[T]]` ma, in questo caso, contenente un solo elemento.
In questo modo le sorgenti che mandano messaggi di tipo `Status[T]` (sensori e attuatori) vengono processate allo stesso modo di quelle che mandano messaggi di tipo `Statuses[T]`, mantenendo la giusta semantica del tipo T, e cioè il tipo di dato rilevato dal sensore/elaborato dal gruppo.

### Group
La classe astratta `Group[I,O]` e le sue implementazioni costituiscono il nucleo della configurazione del gruppo di lavoro. Al suo interno sono presenti sia i campi per la conservazione degli input da trattare, che la funzione higher order definita dall'utente in fase di creazione per trattarli.
Una sua istanza rappresenta lo stato dell'attore, che ne richiama i metodi in risposta ai messaggi che riceve.
In particolare, `insert(author: Actor, newValues: List[I])` viene richiamato per inserire nella mappa sorgente -> valori il nuovo valore ricevuto con un messaggio di tipo `Statuses[I]`
La struttura dati usata per il salvataggio dei dati è una `Map[Actor, List[I]]`, con `Actor` type alias di `ActorRef[Message]`, rendendo quindi banale sovrascrivere i dati in caso di ricezione di più status nel tempo da parte di una sorgente.
Il metodo `compute()` viene richiamato dall'attore nel momento in cui sono state soddisfatte le condizioni di innesco ("sono stati collezionati gli input di tutte le sorgenti" nella versione bloccante o "è arrivato un messaggio contenente un input" in quella non bloccante) e il suo compito è essenzialmente quello di calcolare l'output del gruppo, impostandolo come status (cioè il campo ereditato da `Device`) e, a seconda dell'implementazione, resettare la mappa dei valori parziali.
Questo è il metodo di cui le due implementazioni fornite con la libreria fanno l'override:
* ReduceGroup applica tramite una `foldLeft` la funzione di riduzione `f: I => O` su tutta la lista di liste appiattita contentente tutti i valori di input, producendo un singolo valore di output.
* MapGroup esegue l'override di `Group` tramite un mixin con il trait `MultipleOutput`, che sovrascrive il metodo `propagate()` di `Device` per fare in modo che venga inviato un messaggio di tipo `Statuses[T]` invece che `Status[T]`. Questo perché il MapGroup non fa altro che applicare la funzione di mapping `f: I => O` a ogni elemento della lista di liste appiattita tramite una for comprehension, restituendo in uscita una lista con un numero di elementi variabile ma comunque dipendente in parte dal numero degli elementi processati (e non fissato a uno come nel caso del  `ReduceGroup`).

## Sub-package: Tagging
Questo sottopackacge rappresenta un'estensione del package `grouping` che consente all'utente di inizializzare gruppi di lavoro con un domain specific language, utilizzando i metodi di deployment messi a disposizione dal modulo apposito. I gruppi di base infatti richiedono una creazione degli oggetti potenzialmente molto verbosa che mal si adatta agli obbiettivi di user experience prefissati per la libreria. Per inizializzare gruppi di lavoro senza l'ausilio di questo sottomodulo è infatti necessario possedere già alla creazione del gruppo tutte le `ActorRef` delle sorgenti che fanno parte di quel gruppo. Essendo possibile innestare gruppi uno dentro l'altro, questo requisito rischia di rendere la creazione di sistemi complessi inutilmente articolata.
Per questo motivo in questo modulo è stato fatto largo uso del pattern Factory, attraverso cui vengono generate delle istanze di `Group` rendendo trasparente all'utente l'ordine di generazione dei dispositivi sul cluster.
Essendo un domain-specific language creato per effettuare il deployment di quanto già presente nel modulo di grouping, è inevitabilmente legato alle classi e alle implementazioni lì presenti, lasciando all'utente l'onere di estendere nel modo giusto anche questo modulo in caso abbia esteso in qualche modo `grouping`.

![](PPS-Tagging.png)

### Tag
Un `Tag` costituisce una configurazione per la generazione di una corrispettiva istanza di `Group` tramite il pattern factory durante il deployment.
La creazione di un tag riduce la dipendenza temporale rispetto a quella di un `Group` perchè utilizza gli `ActorRef` di akka soltanto per definire il campo `destinations`. Campo che può essere modificato anche a runtime tramite l'invio di un messaggio di `Subscribe` da parte di un'eventuale nuova destinazione.
Un istanza di `Tag` viene generata automaticamente in base ai valori passati alla funzione `apply()` del suo companion object, andando a costituire una specie di factory nella factory. I parametri sono:
* La lista di id (di tipo stringa) dei dispositivi che costituiranno le sorgenti di dati del gruppo.
* La lista di destinazioni iniziali (di tipo `ActorRef[Message]`) dello status computato dal gruppo.
* La sola funzione f di mapping nel caso si voglia creare un `MapTag`.
* La funzione f di riduzione e il valore iniziale dell'accumulatore nel caso si voglia creare un `ReduceTag`.
* Il tipo di innesco della computazione (bloccante/ non bloccante), rappresentato come caso dell'enum `TriggerMode`.
`MapTag` e `ReduceTag` sono case classes concrete che implementano il trait `Tag` effettuando l'override del metodo `generateGroup(sources: ActorList)`, richiamato dal deployer nel momento in cui viene eseguita la vera e propria factory del gruppo, ovvero quando gli attori di tutti i dispositivi i cui id erano contenuti nella lista delle sorgenti del `Tag` sono stati istanziati con successo (rendendo possibile la costruzione della lista di `ActorRef[Message]` in input).
Il metodo `generateGroup(sources: ActorList)` restituisce come output un'istanza del gruppo parametrizzata secondo i valori con cui è stato creato il `Tag`, effettuando un mixin con il trait `Deployable`, spiegato nel paragrafo seguente.

### Deployable

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

![Storage package classes UML](https://i.imgur.com/otYr8Ua.png)

Il trait `TusowService` è l'interfaccia generata dallo schema protobuf, la quale viene implementata dalla classe `TusowAkkaService`; quest'ultima filtra le richieste in base alla tipologia di *tuple space* e delega l'elaborazione alle classi `TusowAkkaTextualHandler` (in caso di tuple space testuali) e `TusowAkkaLogicHandler` (in caso di tuple space logici).  
La classe `TusowServer` si occupa di invece di spawnare un nuovo nodo da aggiungere al cluster predefinito; questo nodo avrà come figlio un attore non tipizzato di *Akka* il cui *behavior* è dato da una nuova istanza di `TusowAkkaService`.  
In questo modo gli attori del cluster possono chiamare le *procedure* di Tusow interagendo col neonato attore.  

![Sequence diagram of TuSoW](https://i.imgur.com/82MH7vr.png)

## Struttura del progetto

La struttura dei package del progetto è la seguente:

![Packages UML](https://i.imgur.com/YOS8yNq.png)