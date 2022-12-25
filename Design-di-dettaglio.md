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

![Actuator UML](https://i.imgur.com/Ybv1jdD.png)

La classe `Actuator` è un'implementazione del *trait* `Device`. Come tale può venir istanziato con tutti i mixin disponibili per il `trait Device`, ad esempio `Public`.
```scala
new Actuator[String]("actuatorTest", fsm) with Public[String]
```
Un attuatore può, in base al dispositivo fisico che rappresenta e quindi ad eventi esterni, avere stati differenti; questi stati vengono forniti al costruttore tramite una macchina a stati finiti (classe `FSM`): ogni stato della macchina è un'implementazione del trait `State` ed i pesi degli archi sono i messaggi (ovvero gli eventi) che causano un cambiamento di stato. Ad esempio, si prende come ipotesi l'aver fornito una FSM all'attuatore come quella nell'immagine sottostante:

![FSM di esempio](https://i.imgur.com/WJTiX1A.png)

Se l'attuatore si trovasse nello stato **A** e ricevesse un messaggio *GoTo("B")* allora potrebbe spostarsi nello stato **B**, ma se ricevesse un messaggio *GoTo("D")*, poichè esso non è definito nella FSM, il cambio di stato non verrebbe effettuato.  
In questo scenario abbiamo utilizzato degli stati privi di comportamenti, ovvero dei `BasicState`, tuttavia è possibile utilizzare stati il cui cambiamento è legato a condizioni sull'evento (`ConditionalState`) oppure a dei timer periodici (`TimedState`).
Per un uniformare i messaggi scambiati fra i vari attori sia è creato il trait `Message` con varie case class che lo implementano, ad esempio `case class Approved() extends Message`; poichè è necessario che gli utilizzatori finali del framework possano inviare qualunque tipo di oggetto, sono stati creati messaggi che possono incapuslare qualunque tipo utilizzando i generics; ne è un esempio il messaggio `case class MessageWithReply[T](message: T, replyTo: ActorRef[Message], args: T*) extends Message`.

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