# akka-wamp 0.3.0
[![Build Status][travis-image]][travis-url] [![Codacy Status][codacy-image]][codacy-url]

Akka Wamp is a WAMP - [Web Application Messaging Protocol](http://wamp-proto.org/) implementation written in [Scala](http://scala-lang.org/) with [Akka](http://akka.io/)

It actually provides the following features:

|                       | Broker | Publisher | Subscriber | Dealer | Callee | Caller |
|:---------------------:|:------:|:---------:|:----------:|:------:|:------:|:------:|
|         Basic Profile |    X   |     X     |      X     |        |        |        |
|      Advanced Profile |        |           |            |        |        |        |
|    JSON Serialization |    X   |     X     |      X     |        |        |        |
| MsgPack Serialization |        |           |            |        |        |        |
|   WebSocket Transport |    X   |     X     |      X     |        |        |        |
|     Raw TCP Transport |        |           |            |        |        |        |


# Client
Akka Wamp provides you with three alternative APIs in writing clients:

 * future based,
 * actor based,
 * and stream based.


## Future based
Working in progress


## Actor based
If you wish your client to be stateful and thread safe, then Akka Wamp provides you with an [Akka Actor](http://doc.akka.io/docs/akka/2.4/scala/actors.html) based API. You can write an client actor, meaning that all operations are implemented with message passing instead of direct method calls and you'll get all the benefits of _"actor based programming model"_

This approach gives you a lot of flexibility but requires you to have a fair knowledge of the exact messages that can be sent to (and received from) the underlying transport. You're required to know a bit more about the protocol itself.


### Connect a transport
Akka Wamp is built as an [Akka I/O](http://doc.akka.io/docs/akka/2.4/scala/io.html) extension. You can easily connect a transport as you can read from the following snippet:

```scala
import akka.actor._
import akka.io._
import akka.wamp._

object ClientApp extends App {
  import Wamp._
  import messages._

  class Client extends Actor 
  with ActorLogging with SessionScope {
  
    var transport: ActorRef = _
  
    def receive: Receive = {
      case Connected(transport) =>
        this.transport = transport 
        // open a session 
    }
}
```

``Wamp`` is the protocol extension identifier you pass to the Akka ``IO`` entry point to obtain the ``WampManager`` actor reference. Then you send the ``Connect`` message to the manager which takes care of connecting to the remote router at given ``url``. Upon connection, your client receives the ``Connected`` message from the manager with the transport actor reference to be held for later usage.


### Open a session
As stated by the protocol, the first message the client sends upon connection is an ``Hello`` message to request a new session to be opened:


```scala
var transport: ActorRef = _
var sessionId: Long = _

def receive: Receive = {
  case Connected(transport) =>
    this.transport = transport
    transport ! Hello
  
  case Welcome(sessionId, _) =>
    this.sessionId = sessionId
    context become opened
    
  case Abort =>
    this.sessionId = 0
}

def opened: Receive = {
  case Goodbye =>
    transport ! Goodbye("wamp.error.goodbye_and_out")
    
  // case ...  
}

```

Upon session opened, your client receives the ``Welcome`` message from the remote router with the session identifier, which could be held by your client for later usage. At this point, your client could _"hotswap"_ its message loop at runtime (refer to [Akka Actor Become/Unbecome](http://doc.akka.io/docs/akka/2.4.8/scala/actors.html#become-unbecome)) to focus on receiving those messages that could be sent by the remote router while the session is kept open (for example the ``Goodbye`` message) 


### Subscribe to topics
Subscribing to a topic is no more than sending a ``Subscribe`` message to the transport actor. When to send such a message is, of course, up to your application: it could be on processing the ``Welcome`` message or it could be upon receiving any other message you design yourself for your application purposes.

```scala
def receive: Receive = {
  // case Connected ...
  
  case Welcome(sessionId, details) =>
    context become opened
    transport ! Subscribe(nextId, "myapp.early.topic")
}

val requestId: Long = _
val subscriptionId: Long = _

def opened: Receive => {
  case myapp.SubscribeToTopic =>
    this.requestId = nextId
    transport ! Subscribe(requestId, "myapp.late.topic")
  
  case Subscribed(requestId, subscriptionId) =>
    if(requestId == this.requestId) 
      log info s"Subscription to late topic confirmed" 
    
  case e @ Event(subscriptionId, publicationId, details, Some(payload)) =>
    //if (subscriptionId == this.subscriptionId)
    println payload.toMap
}
```

You could subscribe to as many topics as you want and upon subscription, your client receives the ``Subscribed`` message from the remote router with both the request and subscription identifiers. From this point on, your client is able to receive ``Event`` messages from the remote router. You could use the subscription identifier to understand from which topic the event is coming from.


### Publish events
Publishing an event is no more than sending a ``Publish`` message to the transport actor. When to send such a message is up to your application: it could be on processing the ``Welcome`` message or it could be upon receiving any other message you design yourself for your application purposes (for example a ``"tick"`` message being sent by the [Akka Scheduler](http://doc.akka.io/docs/akka/2.4.8/scala/scheduler.html) is a nice example).

```scala
import context.dispatcher
import scala.concurrent.duration._

val tick =
  context.system.scheduler.schedule(500 millis, 1000 millis, self, "tick")
    
def receive: Receive = {
  // case Connected ...
  
  case Welcome(sessionId, details) =>
    context become opened
    transport ! Publish(nextId, "myapp.early.topic") //, Some(payload))
}


def opened: Receive => {
  case "tick"  =>
    // publish at every tick
    transport ! Publish(nextId, "myapp.ticking.topic")
    
  // case ...  
}
```

If your wish to send some payload with data arguments then [read below](#publish).


### Identifiers
As stated by the [protocol](https://tools.ietf.org/html/draft-oberstet-hybi-tavendo-wamp-02#section-5.1.2), any identifier must be generated within scopes which restrict the set of possible values and requires a specific generation strategy to be performed.  

Identifiers for messages originating by clients MUST belong to the so called _"session scope"_ (which also means making sure they are incremented by 1). To be compliant with that requirement you can mixin the ``akka.wamp.SessionScope`` trait and invoke the ``nextId`` method to generate the right identifier. 

> NOTE: there's an open [issue#231](https://github.com/wamp-proto/wamp-proto/issues/231) on GitHub about better identifiers scopes. Akka Wamp is already implementing that proposal.


## Stream based
Working in progress.


# Messages
This sections gives you some details about the protocol to let you quickly understand which messages you're expected to send and receive.

## Transport and Session

### Connect
It is the message you send to the Akka IO entry point to connect a transport to the remote router. It can be constructed passing the following parameters:

* ``client``  
  It is your client actor reference to which messages from the remote router are delivered while the connection is established.
  
* ``url``  
  It is the [URL](https://www.ietf.org/rfc/rfc1738.txt) to the remote router. By default Akka Wamp makes use of ``ws://127.0.0.1:8080/ws`` and it currently supports the following protocols:
  
  * ``ws``  
    It is the WebSocket protocol.
  
* ``subprotocol``  
  It is the [subprotocol](https://www.iana.org/assignments/websocket/websocket.xml#subprotocol-name) you wish to negotiate with the remote router. By default, Akka Wamp sends ``wamp.2.json`` but it supports all of the followings:
  
  * ``wamp.2.json``

Examples:

```scala
IO(Wamp) ! Connect(client)
IO(Wamp) ! Connect(client, "ws://router.host.net:8080/path/to/ws")
IO(Wamp) ! Connect(client, subprotocol = "wamp.2.mgspack")
```


### Hello
It is the message you send to the transport to request a new session being opened with the given realm attached plus additional details. It can be constructed passing the following parameters:

* ``realm``  
  It is the realm identifier given as [URI](https://tools.ietf.org/html/draft-oberstet-hybi-tavendo-wamp-02#section-5.1.1). By default, Akka Wamp sends ``akka.wamp.realm``
  
* ``details``  
   It is a dictionary of additional details. By default, Akka Wamp makes a dictionary with all possible client roles its supports:
    
    * ``subscriber``
    * ``publisher``
   
Examples:

```scala
transport ! Hello()
transport ! Hello("myapp.realm")
transport ! Hello(details = Dict().withRoles("subscriber"))
transport ! Hello("myapp.realm", Dict().withRoles("publisher"))
```
  

### Abort
TBD

### Welcome
It is the message you receive from the remote router upon session opening. You can pattern match the following parameters:

* ``sessionId``  
  It is the session identifier as generated by the remote router.
  
* ``details``  
  It is a dictionary with additional details (for example the remote router agent identifier)
  
Examples:

```scala
val sessionId: Long = _

def receive: Receive = {
  case Welcome(sessionId, details) =>
    log info s"Session $sessionId opened $details"
    this.sessionId = sessionId
    context become opened
}

def opened: Receive = {
  case Goodbye =>
    this.sessionId = 0
    
  // case ...  
}
```


### Goodbye
It is the message either you send to the transport or receive from the remote router to request session closing. It can be constructed passing the following parameters:

* ``reason``  
  It is the reason identifier given as [session close URI](https://tools.ietf.org/html/draft-oberstet-hybi-tavendo-wamp-02#section-10.1.3). By default, Akka Wamp sends ``wamp.error.close_realm``
  
* ``details``  
  It is a dictionary of additional details. By default, Akka Wamp sends an empty dictionary.
   
Some sending snippets are:

```scala
transport ! Goodbye()
transport ! Goodbye("wamp.error.system_shutdown")
transport ! Goodbye(details = Dict().withEntry(reason -> "Serious error occured!"))
```

A receiving snippet is:

```scala
def opened: Receive = {
  case Goodbye(reason, details) =>
    log warn s"Router closed session because of $details"
    transport ! Goodbye("wamp.error.goodbye_and_out")
    if (reason != "wamp.error.system_shutdown")
      // why not requesting a new session ;-)
      transport ! Hello
    else
      IO(Wamp) ! Connect(self)
}
```

> NOTE: there's an open [issue#242]((https://github.com/wamp-proto/wamp-proto/issues/242) on GitHub about the ``Goodbye`` message to warn you that some remote routers could decide to disconnect the transport in addition to close the session. If that happens the you'll be forced to reconnect.

## Subscriber and Publisher 

### Subscribe
It is the message you send to the transport to subscribe to a topic. It can be constructed passing the following parameters:

* ``requestId``
  It is the request identifier you have to generate in _"session scope"_. AkkaWamp provides you with a ``SessionScope`` trait you can mixin to invoke ``nextId``

* ``topic``  
  It is the topic URI you want to subscribe to.
  
* ``options``  
  It is a dictionary of additional details. By default, Akka Wamp sends an empty dictionary. 

Examples:
  
```scala
transport ! Subscribe(requestId = 34, "myapp.tick.topic")


### Subscribed
TBD

### <a name="publish"></a>Publish
It is the message you send to the transport to publish an event. It can be constructed passing the following parameters:

* ``requestId``  
  It is the request identifier you have to generate in _"session scope"_. AkkaWamp provides you with a ``SessionScope`` trait you can mixin to invoke ``nextId``
  
* ``topic``  
  It is the topic URI to which you wish to publish to
  
* ``payload``  
  It is the optional payload you could provide to send data arguments. By default, Akka Wamp sends none but you could construct some passing in any arguments. You could even pass key-value pairs alongside values. In that case Akka Wamp generates the missing keys as ``arg0``, ``arg1``, ``arg2``, etc.
  
* ``options``  
  It is a dictionary of additional details (for example with the ``"acknowledge" -> true`` entry if you wish to receive the ``Published`` message from the router). By default, Akka Wamp sends an empty dictionary.
  
Examples:
  
```scala
transport ! Publish(requestId = 34, "myapp.tick.topic")

val list = Payload("paolo", 40, true)
transport ! Publish(nextId, "myapp.topic1", Some(payload))
// serialize to ["paolo", 40, true]

val mixed = Payload("paolo", "age"->40, true)
transport ! Publish(nextId, "myapp.topic1", Some(mixed))
// serialize to {"arg0": "paolo", "age": 40, "arg2": true}

transport ! Publish(89, "myapp.topic3", options = Dict().withAcknowledge(true))
```

### Error
TBD
  

### Event
It is the message you receive from the remote router each time other clients have published to the same topics you subscribed to. You can pattern match the following parameters:

* ``subscriptionId``    
  It is the subscription identifier you could use to figure out which topic the event has been fired for.
  
* ``publicationId``    
   It is the publication identifier generated by the remote router in global scope. 

* ``payload``    
  It is the payload with application data. It could be ``Some(payload)`` or ``None``. When ``Some(payload)`` you shall expect a ``List(arguments)`` of any value. 
     
* ``details``  
  It is a dictionary with additional details.
  
Examples:

```scala
val subscriptionId: Long = _

def opened: Receive = {
  case Event(subscriptionId, publicationId, None, details) =>
    log debug s"Event received for subscription $subscriptionId"
    
  case Event(_, _, Some(payload), _) =>
    log debug s"Event receive with payload $payload"
}
```


# Serialization
TBD


# Router
Akka Wamp also provides a router that can be either embedded into your application or launched as standalone server process.

## Embedded
Make your SBT build depend on akka-wamp:

```scala
scalaVersion := "2.11.8"

libraryDependencies ++= Seq(
  "com.github.angiolep" %% "akka-wamp" % "0.3.0"
  // ...
)
```

Create both and actor system and materializer, and then create the router actor as follows:

```scala
import akka.actor._
import akka.wamp.router._

implicit val sys = ActorSystem("wamp")
implicit val mat = ActorMaterializer()
sys.actorOf(Router.props(), "router")
```

It will automatically bind on a server socket by reading the following Akka configuration

 - ``akka.wamp.protocol``  
   The protocol the router uses as transport (default is ``ws`` WebSocket)
 
 - ``akka.wamp.subprotocol``  
    The subprotocol the router uses when transport is WebSocket (default is ``wamp.2.json``)
 
 - ``akka.wamp.iface``  
   The network interface the router binds to (default is ``127.0.0.1``)
    
 - ``akka.wamp.port``  
   The port number router binds to (default is ``8080``)
 
 - ``akka.wamp.path``  
   The path the router waits for WebSocket connections (default is ``/ws``)
    
> NOTE: because of the default configuration, the Akka Wamp router will expect WebSocket connections at the URL ``ws://127.0.0.1:8080/ws``
   
 

## Standalone
Download and launch the router as standalone application:

```bash
curl https://dl.bintray.com/angiolep/universal/akka-wamp-0.3.0.tgz
tar xvfz akka-wamp-0.3.0.tar.gz
cd akka-wamp-0.3.0
./bin/akka-wamp -Dakka.loglevel=DEBUG
```

You can ovveride the default setting by passing Java system properties on the command line.


# Limitations

 * It works with Scala 2.11 only.
 * It provides WebSocket transport only without SSL/TLS encryption.  
 * The router works as _broker_ only (_dealer_ is NOT provided yet).
 * The client works as _publisher_ and _subscriber_ only (_callee_ and _caller_ are NOT provided yet).
 * It implements the WAMP Basic Profile only (WAMP Advanced Profile is NOT provided yet)
 

[travis-image]: https://travis-ci.org/angiolep/akka-wamp.svg?branch=master
[travis-url]: https://travis-ci.org/angiolep/akka-wamp

[codacy-image]: https://api.codacy.com/project/badge/grade/f66d939188b944bbbfacde051a015ca1
[codacy-url]: https://www.codacy.com/app/paolo-angioletti/akka-wamp
