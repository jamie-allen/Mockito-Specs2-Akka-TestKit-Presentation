!SLIDE title-page

Akka TestKit, Specs2 and Mockito

Jamie Allen

<img src="typesafe-logo-081111.png" class="illustration" note="final slash needed"/>

!SLIDE transition=blindY
# Agenda

* Akka TestKit
* Specs2
* Mockito
* See how they come together

!SLIDE bullets incremental transition=blindY
.notes Akka is written to prevent blocking behaviors as much as possible.  You can still write code inside of actors that blocks, or have logic that looks synchronous when sending messages to actors.
# Akka TestKit

* A testing framework for Akka actors
* Can handle "blocking" and asynchronous tests

!SLIDE transition=blindY
.notes Creating blocking actors is harder in Akka 2.0 from a message-passing perspective, but actors can still encapsulate blocking, side-effecting behaviors (calling to a database, putting data in an external message queue, etc).  Wrap those behaviors in future calls with timeouts.
# Angry Viktor

* Please do not write blocking actors
* You will make Viktor Klang angry
* You won't like Viktor Klang when he's angry

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
# Non-determinism in Testing

* Verifyable results cannot be expected to occur in a specific order
* Accept some level of probabilistic failure
* Focus on a side effect you require, verify that it occurs eventually

!SLIDE bullets incremental transition=blindY
.notes Stop/restart shut down the actor and its message queue, suspend/resume just pause it.  Watch/unwatch are for actor linkages, receiving a terminated message if an actor dies.  compareTo is done by address, allows you to see if an underlying actor ref is the same as one before it should have died.  If isTerminated returns true, you know it will always be true, but if it returns false, it might be not still be alive (race condition)
# TestActorRef

* Allows you to manipulate actors
  * stop/restart
  * suspend/resume
  * watch/unwatch
* Allows you to inspect actor state
  * isLocal
  * isTerminated
  * compareTo

!SLIDE bullets incremental transition=blindY
.notes Actors just don't respond to messages, they can become/unbecome.  NEVER try to call methods on your actor except via passing messages to the mailbox.  Don't mark them private, as that means a synthetic public method has to be created to call into the method from the receive closure.  All "private" methods should be final, for performance optimization reasons.  They would also be exposed by TestActorRef, and ActorRef should in general mask any ability to call those methods.  But the real problem is that if someone does get a reference to your actor, they can call into the actor with another thread, which introduces the very concurrency issues you're trying to avoid by using actors!
# Why TestActorRef?

* Uncertain behavior and responses of Actors
* Allows direct access to the underlying reference

!SLIDE transition=blindY
.notes Gives you more flexibility to interact with actors using testing tools and frameworks that are more geared to traditional classes.  But why would you do this?  The reason is to test expectations that an actor will throw the correct exception if something illegal occurs, from a unit test perspective.  From an integration perspective, you would want to force the error and then check that the actor is properly restarted based on your defined strategy.  Note that you now have a direct reference to that specific instance of the actor - if the actor is restarted, you do not have a reference to the new instance wrapped in that TestActorRef.  However, this is useful if you want to perform post-mortem checks on the actor.
# How to Access the Underlying Actor

    val actorRef = TestActorRef[MyActor]
    val actor = actorRef.underlyingActor

    actor.receive("hello")

    actorRef ! PoisonPill

    assertFalse(actorRef.underlyingActor eq actor)

!SLIDE transition=blindY
# An Example of a Direct Call to Test Exceptions
   
    val actorRef = TestActorRef(new Actor {
      def receive = {
        case boom ⇒ throw new IllegalArgumentException("boom")
      }
    })

    // NOTE: You don't need the underlyingActor() call on the actorRef
    intercept[IllegalArgumentException] { actorRef.receive("hello") }

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
# Bad TestKit Test - What is Wrong Here?

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
# Specs2

* Behavior-Driven Development testing framework
* Create tests with intentions that are clear to devs and stakeholders
* Based on natural language

!SLIDE bullets incremental transition=blindY
.notes The only public interface of an actor is its mailbox, but the API consists of messages it can handle
# Unit Tests

* Test names are sentences starting with the word "should"
* Only test the APIs of a single entity

!SLIDE bullets incremental transition=blindY
# Integration/Acceptance Tests

* Acceptance test names are written using an agile "User Story"
* "As a [role]", I want [feature] so that [benefit]""
* "Given [initial context], when [event occurs], then [ensure some outcome]"

!SLIDE bullets incremental transition=blindY
.notes Created in the model of earlier frameworks, such as EasyMock and JMock.  Test spy is a model that allows you to verify behavior and stub methods, but also allows you to embed assertions without external calls.
# Mockito

* Technically, a "Test Spy Framework"
* Verify behaviors 
* Stub methods or classes that haven't been implemented yet
* Stub dependencies that are external to the test being executed

!SLIDE transition=blindY
.notes You had to manage your anonymous implementations to show that something occurred, or that it happened the correct number of times. This doesn't look TOO bad below, but if you had interfaces/traits that are very large (a smell test in itself), or had to use external libraries with large interfaces/traits that you couldn't control, it got out of hand very quickly.
# Why Use Mockito?

    // Some interface on which the class we're testing depends
    trait MyInterface {  
      def testCall: String
    }

    // Anonymous impl allowing us to verify results
    val myImpl = new MyInterface() { 
      var _callCount = 0
      def callCount = _callCount
      override def testCall() = {
        callCount += 1
        "Here"
      }
    }

    myImpl.callCount // res0: Int = 0

    // How to verify expected results
    assertTrue(myImpl.testCall == "Here")
    assert(myImpl.callCount == 1) // res2: Int = 1

!SLIDE bullets incremental transition=blindY
# Benefits of Mockito

* Replaces the need for anonymous implementations of interfaces/traits
* Can stub classes and interfaces/traits with the same API
* Separates logic of expectations with the verification of results

!SLIDE transition=blindY
# The Same Logic with Mockito

    // Some interface on which the class we're testing depends
    trait MyInterface {  
      def testCall: String
    }

    import org.mockito.Mockito._
    val myMockImpl = mock[MyInterface]

    when(myMockImpl).thenReturn("Here")
    assertTrue(myMockImpl.testCall == "Here")
    verify(myMockImpl, atLeastOnce).testCall

!SLIDE bullets incremental transition=blindY
.notes Static methods would be those on Objects.  Important to avoid shared mocks to avoid thread safety issues if tests are executed in parallel
# Things Not To Do with Mockito

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

* [Testing Actor Systems (Scala)](http://doc.akka.io/docs/akka/2.0/scala/testing.html#integration-testing-with-testkit)
* [Specs2 User Guide](http://etorreborre.github.com/specs2/guide/org.specs2.UserGuide.html#User+Guide)
* [Wikipedia on BDD](http://en.wikipedia.org/wiki/Behavior_Driven_Development)
* [Mockito](http://code.google.com/p/mockito/)
* [Maciej Matyjas - Testing Blocking Messages with Akka's TestKit](http://maciejmatyjas.com/2012/02/23/testing-blocking-messages-with-akkas-testkit/)