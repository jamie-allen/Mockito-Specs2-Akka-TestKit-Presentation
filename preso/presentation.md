!SLIDE title-page

Akka TestKit, Specs2 and Mockito

<img src="typesafe-logo-081111.png" class="illustration" note="final slash needed"/>

!SLIDE bullets incremental transition=blindY
.notes Deterministic with respect to order of events and no concurrency concerns.
# Agenda

* Akka TestKit
* Specs2
* Mockito
* See how they come together

!SLIDE bullets incremental transition=blindY
.notes Creating blocking actors is harder in Akka 2.0 from a message-passing perspective, but actors can still encapsulate blocking, side-effecting behaviors (calling to a database, putting data in an external message queue, etc). Wrap those behaviors in future calls with timeouts!
# Akka TestKit

* A testing framework for Akka actors
* Can handle blocking and asynchronous tests
* Please do not write blocking actors
	* You will make Viktor Klang angry
	* You won't like Viktor Klang when he's angry

!SLIDE transition=blindY
# Angry Viktor
<img src="viktor.png" class="illustration" note="final slash needed"/> => <img src="237035-hulk.jpg" class="illustration" note="final slash needed"/>

!SLIDE bullets incremental transition=blindY
.notes Deterministic with respect to order of events and no concurrency concerns.
# Actor Testing Models

* Unit Test
	* Isolated
	* Single-threaded
	* Deterministic
* Integration Test
	* Multiple and Encapsulated
	* Multithreaded
	* Non-deterministic

!SLIDE bullets incremental transition=blindY
.notes Actors just don't respond to messages, they can become/unbecome
# Why TestActorRef?

* Uncertain behavior and responses of Actors
* ActorRef shields actors, only interaction by messages/mailboxes
* TestActorRef also exposes the underlying reference
* TestActorRef also supports querying the actor's receive

!SLIDE transition=blindY
.notes Gives you more flexibility to interact with actors using testing tools and frameworks that are more geared to traditional classes.
# How to Access the Underlying Actor

	val actorRef = TestActorRef[MyActor]
	val actor = actorRef.underlyingActor

	actor.receive("hello") // <- NOT A MESSAGE, NOT IN MAILBOX!

!SLIDE transition=blindY
.notes This just sends a message to an actor and expects a message of a specific type back
# Simple TestKit Test

	"A MyActor" should "respond asynchronously to a message" in {
      val testDuration = Duration(2, SECONDS)
      implicit val timeout = Timeout(testDuration)

      val myActorRef = TestActorRef[MyActor]

      myActorRef ! DataQuery("foo", "bar")

      expectMsgClass(testDuration, classOf[MyResponse])
    }

!SLIDE transition=blindY
.notes What is wrong with this test?  This example has no way to capture the result of the future call
# Bad TestKit Test

    "A MyActor" should "receive a response to this message" in {
      val testDuration = Duration(2, SECONDS)
      implicit val timeout = Timeout(testDuration)

      val myActorRef = TestActorRef[MyActor]

      myActorRef ? MyMessageToGetSomeData

      expectMsgClass(testDuration, classOf[MyResponse])
    }

!SLIDE transition=blindY
# Good TestKit Test

    "A MyActor" should "receive a response to this message" in {
      val testDuration = Duration(2, SECONDS)
      implicit val timeout = Timeout(testDuration)

      val myActorRef = TestActorRef[MyActor]

      val response = Await.result(myActorRef ? 
                       DataQuery("foo","bar"),
                         testDuration).asInstanceOf[MyResponse]

      response should not be ('empty)
    }

!SLIDE bullets incremental transition=blindY
.notes You had to manage your anonymous implementations to show that something occurred, or that it happened the correct number of times.
# Mockito

    trait MyInterface {
      def testCall
    }

    val myImpl = new MyInterface() {
      var _callCount = 0
      def callCount = _callCount
      override def testCall() = callCount += 1
    }

    myImpl.callCount // res0: Int = 0
    myImpl.testCall
    myImpl.callCount // res2: Int = 1

!SLIDE bullets incremental transition=blindY
. notes Created in the model of earlier frameworks, such as EasyMock and JMock
# What is Mockito?

* Technically, a "Test Spy Framework"
* Verify behaviors 
* Stub methods

!SLIDE bullets incremental transition=blindY
# Benefits of Mockito

* Replaces the need for anonymous implementations of interfaces/traits
* Can stub classes and interfaces/traits with the same API
* Separates logic of expectations with the verification of results

!SLIDE bullets incremental transition=blindY
.notes Static methods would be those on Objects.  Important to avoid shared mocks to avoid thread safety issues if tests are executed in parallel
# Things NOT To Do with Mockito

* Do not mock final classes
* Do not mock final methods
* Do not mock static methods
* Do not mock equals() or hashCode()
* Cannot mock private methods
* Avoid "shared" mocks

!SLIDE transition=blindY
# Example of Shared Mock


!SLIDE bullets incremental transition=blindY
.notes Depends on objenesis, which may not work on J9. The limitation of objenesis should only be a production concern?
# Things to Note About Mockito

* Might not be supported on all VMs
* Only use it to mock dependencies via a service interface

!SLIDE transition=blindY
# References

* [Mockito](http://code.google.com/p/mockito/)
* [Specs2 User Guide](http://etorreborre.github.com/specs2/guide/org.specs2.UserGuide.html#User+Guide)
* [Testing Actor Systems (Scala)](http://doc.akka.io/docs/akka/2.0/scala/testing.html#integration-testing-with-testkit)
* [Maciej Matyjas - Testing Blocking Messages with Akka's TestKit](http://maciejmatyjas.com/2012/02/23/testing-blocking-messages-with-akkas-testkit/)