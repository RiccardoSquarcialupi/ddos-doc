# Programmazione funzionale

## Pattern matching

### Matching sull'oggetto

In questo caso il match si comporta in modo analogo ad uno switch di Java ma con alcune funzionalità extra.

```scala
override def validateTupleSpace(in: TupleSpaceID): Future[IOResponse] = in.`type` match {  
    case TupleSpaceType.LOGIC => logicHandler.validateTupleSpace(in)  
    case TupleSpaceType.TEXTUAL => textualHandler.validateTupleSpace(in)  
}
```

### Class decomposition

Utilizzato per destrutturare *case class* ed accedere agli oggetti interni.

```scala
message match  
    case MessageWithoutReply(msg: T, args: _*) =>  
        messageWithoutReply(msg, context, args.asInstanceOf[Seq[T]])  
        Behaviors.same  
    case Approved() =>  
        utilityActor = approved(context)  
        Behaviors.same  
    case Denied() =>  
        denied()  
        Behaviors.same  
    case ForceStateChange(transition: T) =>  
        println(s"Force state change to ${transition}")  
        utilityActor = forceStateChange(context, transition)  
        Behaviors.same  
    case Subscribe(requester: ActorRef[Message]) =>  
        subscribe(context.self, requester)  
        Behaviors.same  
    case PropagateStatus(requester: ActorRef[Message]) =>  
        propagate(context.self, requester)  
        Behaviors.same
```

## Higher-order functions

Le higher-order functions sono un costrutto che permette di utilizzare funzioni come parametri di altre funzioni.

```scala
class ConditionalState[T](name: String, condFunction: (T, Seq[T]) => Boolean) extends State[T](name):  
  
    private val behavior: Behavior[Message] = Behaviors.receiveMessage[Message] { msg =>  
        msg match  
            case MessageWithReply(msg: T, replyTo, args: _*) =>  
                if condFunction(msg, args.asInstanceOf[Seq[T]]) then  
                    replyTo ! Approved()  
                else  
                    replyTo ! Denied()  
            case _ =>  
        Behaviors.same  
    }
```

## Type alias

I type alias permettono di ridefinire una classe od un tipo con un altro nome. Molto utile per associare a delle higher-order functions un stringa più comprensibile.  
Il frammento di codice precedente viene dunque riscritto nel seguente modo:

```scala
type ConditionalFunction[T] = (T, Seq[T]) => Boolean

class ConditionalState[T](name: String, condFunction: ConditionalFunction[T]) extends State[T](name):  
  
    private val behavior: Behavior[Message] = Behaviors.receiveMessage[Message] { msg =>  
        msg match  
            case MessageWithReply(msg: T, replyTo, args: _*) =>  
                if condFunction(msg, args.asInstanceOf[Seq[T]]) then  
                    replyTo ! Approved()  
                else  
                    replyTo ! Denied()  
            case _ =>  
        Behaviors.same  
    }
```

## Options

L'utilizzo di valori `null` è altamente sconsigliata e porta con alta probabilità a scrivere codice buggato. Per gestire la *null safety*, Scala mette a disposizione gli `Option`, strutture per incapsulare oggetti che possono assumere valori `null` a run-time.

```scala
  protected var status: Option[T] = None
  
  def propagate(selfId: ActorRef[DeviceMessage], requester: ActorRef[DeviceMessage]): Unit =
    if requester == selfId then status match
      case Some(value) => for (actor <- destinations) actor ! Status[T](selfId, value)
      case None =>
```

## Mixins

Utilizzando i mixins è possibile comporre classi senza utilizzare il meccanismo dell'ereditarietà.

```scala
trait Condition[I: DataType, O: DataType](condition: O => Boolean, replyTo: ActorRef[DeviceMessage]):

  self: Sensor[I, O] =>
  
  override def update(selfId: ActorRef[SensorMessage], physicalInput: I): Unit =
    self.status = Option(preProcess(physicalInput))
    if condition(self.status.get) then replyTo ! Status[O](selfId, self.status.get)


new Actuator[String]("actuatorTest", fsm) with Condition[String, String]
```

## Type Class
//TODO

## Partial functions

```scala
private def handleReadOrTakeRequest[A](in: ReadOrTakeRequest)(readOrTake: (TextualSpace, String, Long) => Future[A]): Future[A] = {  
    val space = textualSpaces(in.tupleSpaceID.getOrElse(TupleSpaceID("")).id)  
    handleFutureRequest(space)(() => Future.failed(new IllegalArgumentException("Tuple space not found")))(() => readOrTake(space, in.template.textualTemplate.get.regex, timeout))  
}
```

## Generics

I generics sono uno strumento che permette di definire un tipo parametrizzato, che viene esplicitato successivamente in fase di compilazione secondo le necessità; i generics permettono di definire delle astrazioni sui tipi di dati definiti nel linguaggio.

```scala
object Graph:  
  
    def apply[T](): Graph[T] = new Graph[T](None, Map.empty)  
  
    def apply[T](edges: (T, T)*): Graph[T] =  
        new Graph(edges.head.getFirst, edges.foldLeft(Map[T, List[T]]())((map, edge) => addEdge(map, edge)))

case class Graph[T](private val initialNode: Option[T], private var edges: Map[T, List[T]]):

[...]

```

## Symbolic methods

In Scala è possibile utilizzare caratteri speciali per definire metodi e funzioni. Per poter garantire la compatibilità con altri linguaggi basati su JVM (e.g. Java, Kotlin), è possibile annotarli con l'annotazione `@targetName("alternative name")` il cui contenuto verrà utilizzato per rinominare il metodo in fase di compilazione.

```scala
@targetName("forEach")  
def @-> (f: (T, List[T]) => Unit): Unit = edges foreach (x => f(x._1, x._2))

@targetName("getEdgesOrElseEmptyList")  
def ?-> (node: T): List[T] = edges.getOrElse(node, List.empty)

@targetName("containsNode")  
def ?(node: T): Boolean = edges.contains(node)  
  
@targetName("addEdge")  
def ++(edge: (T, T)): Unit = edges = Graph.addEdge(edges, edge)
```


## Programmazione asincrona

Mediante l'uso delle Future è possibile eseguire computazioni asincrone, ovvero delegandole ad entità terze (nell'esempio sottostante ad eseguire sarà l'`ExecutionContext` passato implicitamente).

```scala

def joinFutures[T](futures: Seq[CompletableFuture[T]])(implicit executor: ExecutionContext): Unit =
    Future.sequence(futures.map(_.toScala)).onComplete(_ => ())  


def processReadOrTakeAllFutures[A](futures: Seq[CompletableFuture[A]])(result: A => Tuple)(implicit executor: ExecutionContext): Future[TuplesList] =  
    var tuplesList = scala.Seq[Tuple]()  
    futures.foldLeft(CompletableFuture.allOf(futures.head))(CompletableFuture.allOf(_, _)).toScala.map(_ => {  
        futures.foreach(f => {  
            tuplesList = tuplesList :+ result(f.get())  
        })  
        TuplesList(tuplesList)  
    })
```

## Implicits

I parametri impliciti sono i parametri definiti con la keyword `implicit`, ovvero i parametri che verranno passati alla funzione in basa al contesto in cui viene chiamata.

```scala
def run(): Future[Http.ServerBinding] = 
    implicit val sys = system  
    implicit val ec: ExecutionContext = system.dispatcher  
  
    val service: HttpRequest => Future[HttpResponse] =  
        TusowServiceHandler(tusowAkkaService)
```

## For-comprehensions

Tramite le for-comprehensions di Scala è possibile esprimere sinteticamente algoritmi molto complessi in modo idiomatico al linguaggio.
```scala
val tagTags = for {  
  t <- newTags  
  nestedTag <- t.getTags() if !accumulator.contains((nestedTag -> t.id))  
} yield (nestedTag -> t.id)
```

## Programmazione reattiva

Usando la librearia akka.stream (non importata direttamente ma come dipendenza di akka.gRPC), ci si imbatte nel concetto di *stream*, ovvero di *flusso attivo in cui si spostano e modificano dati*[^1]. Usando la classe `Source` è possibile immettere dentro lo stream nuovi dati generando degli eventi ai quali, medianti dei *listener*, sarà possibile reagire. La gestione degli eventi viene auto-generata dai file *protobuf* usando il compilatore di Akka gRPC.

```scala
override def writeAllAsStream(in: WriteAllRequest): Source[IOResponse, NotUsed] =  
    textualSpaces(in.tupleSpaceID.getOrElse(TupleSpaceID("")).id) match  
        case null => Source.empty  
        case space@_ =>  
            val futures = in.tuplesList.get.tuples.map(t => space.write(t.value))  
            TusowGRPCCommons.joinFutures(futures)  
            Source(futures.map(f => IOResponse(response = true, message = f.get.toString)).toList)
```

## Programmazione logica

Per scrivere e leggere tuple da TuSoW è stato utilizzato Prolog.

```scala
val tuple = new Tuple("", "loves(romeo, juliet).")  
val readTemplate = new Template.Logic("loves(romeo, X).")
val writeResponse = Await.result[IOResponse](client.write(new WriteRequest(Some(tupleSpace), Some(tuple))), Duration(5000, TimeUnit.MILLISECONDS))
val readResponse = Await.result[Tuple](client.read(new ReadOrTakeRequest(Some(tupleSpace), readOrTakeRequestTemplate)), Duration(5000, TimeUnit.MILLISECONDS))
assert(readResponse == "[X = juliet]")
```

[^1]: https://doc.akka.io/docs/akka/current/stream/stream-flows-and-basics.html

# Sezioni personali

## Filippo Cavallari

Durante lo sviluppo del primo modulo mi sono occupato dello sviluppo dell'attuatore (classe `Actuator`) e degli stati (`State`, `BasicState`, `CondState`, `TimedState`). In seguito ho contribuito allo sviluppo del modulo di deployment, implementando la struttura dati `Graph` e il metodo per deployare un grafo di dispositivi. Per concludere ho sviluppato l'intero modulo di storage, integrando TuSoW con Akka gRPC in modo che potesse essere utilizzato all'interno del framework; quest'ultimo sviluppo è quello che si è rivelato essere più complesso ma che a permesso di integrare la programmazione logica con quella funzionale.
Sebbene alcune classi non fossero di mia competenza, duramente alcune situazioni critiche ho contribuito ad implementare alcune funzionalità ed a risolvere alcuni bug bloccanti.

La lista finale delle classi che ho sviluppato in autonomia sono:
* `Actuator`
* `State`
* `BasicState`
* `CondState`
* `TimedState`
* `Graph`
* `LateInit`
* `TusowAkkaService`
* `TusowTextualHandler`
* `TusowLogicHandler`
* `TusowServer`
* `TusowClient`
* `TusowGRPCCommons`

Mentre la lista delle classi a cui ho contribuito parzialmente comprende:

* `Deployer`
* `FSM`
* `Message`
* `Device`

Gli unit test che ho sviluppato comprendono le classi:

* `DeployerTest`
* `GraphTest`
* `ActuatorTest`
* `TusowLogicHandlerTest`

## Andrea Ingargiola

Inizialmente mi sono occupato di mantenere una visione d'insieme nel design dei due componenti principali del modulo `Device`, partecipando alla concezione sia dell'Attuatore (implementando la generica macchina a stati finiti) che del Sensore, e implementando direttamente il protocollo di comunicazione dell'astrazione `Device`, cioè il modulo `Public`.
Dopo di che mi sono dedicato al modulo di `Grouping`, progettando, implementando e testando tutte le classi che lo compongono. Accorgendomi di alcune criticità nella semplicità di utilizzo del modulo sia da solo che in tandem con il modulo `Deployment`, ho deciso di aggiungere il sotto-modulo `Tagging`, che tramite l'utilizzo del pattern factory va a creare un domain-specific language che, oltre a porre rimedio alle lacune di `Grouping`, ha permesso di cimentarmi in funzioni effettivamente complesse da scrivere (`deployGroups`, `retrieveTagSet` e `exploreInnerTags`) e che potessero sfruttare  (e cioè pattern matching, funzioni ricorsive senza side effects, for-comprehension avanzate e uso avanzato delle collezioni).

Le classi progettate e implementate direttamente da me sono:
* `FSM`
* `Public`
* `Subscribe`
* `SubscribeAck`
* `Statuses`
* `Group`
* `MapGroup`
* `ReduceGroup`
* `MultipleOutput`
* `GroupActor`
* `BlockingGroup`
* `NonBlockingGroup`
* `Tag`, `MapTag` e `ReduceTag`
* `Deployable`
* `Taggable`
* `TriggerMode`
* `Deployer` (metodi `deployGroups`, `retrieveTagSet` e `exploreInnerTags`)

Mentre le classi che ho contribuito a progettare sono:
* `Sensor` 
* `Actuator`
* `Device`

Gli unit test che ho sviluppato comprendono le classi:
* `FSMTest` 
* `PublicTest`
* `GroupTest`
* `TagTest`   

