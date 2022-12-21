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

#### Higher-order functions

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

### Type alias

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

### Options

L'utilizzo di valori `null` è altamente sconsigliata e porta con alta probabilità a scrivere codice buggato. Per gestire la *null safety*, Scala mette a disposizione gli `Option`, strutture per incapsulare oggetti che possono assumere valori `null` a run-time.

```scala
protected var status: Option[T] = None  

def propagate(selfId: ActorRef[Message], requester: ActorRef[Message]): Unit =  
  if requester == selfId then status match  
    case Some(value) => for (actor <- destinations) actor ! Status[T](selfId, value)  
    case None =>
```

### Mixins

Utilizzando i mixins è possibile comporre classi senza utilizzare il meccanismo dell'ereditarietà.

```scala
trait Condition[A, B](condition: (A | B) => Boolean, replyTo: ActorRef[Message]): 

  self: Sensor[A, B] =>  
  
  override def update(selfId: ActorRef[Message], physicalInput: B): Unit =  
    self.status = Option(preProcess(physicalInput))  
    condition(self.status.get) match  
      case true => replyTo ! Status[A](selfId, self.status.get)  
      case _ =>


new Actuator[String]("actuatorTest", fsm) with Condition[String, String]
```

### Partial functions

```scala
private def handleReadOrTakeRequest[A](in: ReadOrTakeRequest)(readOrTake: (TextualSpace, String, Long) => Future[A]): Future[A] = {  
    val space = textualSpaces(in.tupleSpaceID.getOrElse(TupleSpaceID("")).id)  
    handleFutureRequest(space)(() => Future.failed(new IllegalArgumentException("Tuple space not found")))(() => readOrTake(space, in.template.textualTemplate.get.regex, timeout))  
}
```


### Programmazione asincrona

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

### Implicits

I parametri impliciti sono i parametri definiti con la keyword `implicit`, ovvero i parametri che verranno passati alla funzione in basa al contesto in cui viene chiamata.

```scala
def run(): Future[Http.ServerBinding] = 
    implicit val sys = system  
    implicit val ec: ExecutionContext = system.dispatcher  
  
    val service: HttpRequest => Future[HttpResponse] =  
        TusowServiceHandler(tusowAkkaService)
```

### Programmazione reattiva

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

[^1]: https://doc.akka.io/docs/akka/current/stream/stream-flows-and-basics.html

# Sezioni personali

## Filippo Cavallari

Durante lo sviluppo del primo modulo mi sono occupato dello sviluppo dell'attuatore (classe `Actuator`) e degli stati (`State`, `BasicState`, `CondState`, `TimedState`). In seguito ho contribuito allo sviluppo del modulo di deployment, implementando la struttura dati `Graph` e il metodo per deployare un grafo di dispositivi. Per concludere ho sviluppato l'intero modulo di storage, integrando TuSoW con Akka gRPC in modo che potesse essere utilizzato all'interno del framework.

La lista finale delle classi che ho sviluppato in autonomia sono:
* `Actuator`
* `State`
* `BasicState`
* `CondState`
* `TimedState`
* `Graph`
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

> Per ogni classe e funzione sviluppata ho implementato il relativo unit test utilizzando `scala-test`.