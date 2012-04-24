!SLIDE title-page

Specs2, Mockito and Akka TestKit

<img src="typesafe-logo-081111.png" class="illustration" note="final slash needed"/>

!SLIDE bullets incremental transition=blindY
.notes You had to manage your anonymous implementations to show that something occurred, or that it happened the correct number of times
# What is mocking?

* Used to be anonymous implementations of interfaces
* Mocking frameworks like EasyMock and JMock

!SLIDE bullets incremental transition=blindY
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

!SLIDE bullets incremental transition=blindY
.notes Depends on objenesis, which may not work on J9. The limitation of objenesis should only be a production concern?
# Things to Note About Mockito

* Might not be supported on J9
* Only use it to mock dependencies via a service interface

!SLIDE bullets incremental transition=blindY
# Akka TestKit

* A testing framework for Akka actors
* Can handle blocking and asynchronous tests
* Please do not write blocking actors.  
	* You will make Viktor Klang angry.  
	* You won't like Viktor Klang when he's angry.

!SLIDE transition=blindY
# Angry Viktor

<img src="237035-hulk.jpg" class="illustration" note="final slash needed"/>

!SLIDE transition=blindY
.notes There is no way to capture the result of the future call
# Bad TestKit Test

    // This code APPEARS to block on response, but the ? is a FUTURE
    "This actor test" should "this return a message" in {
      val testDuration = Duration(2, SECONDS)
      implicit val timeout = Timeout(testDuration)

      val myActorRef = system.actorOf(Props[MyActor])

      myActorRef ? MyMessageToGetSomeData

      expectMsgClass(testDuration, classOf[List[MyReturnData]])
    }

!SLIDE transition=blindY
# Good TestKit Test

    // This code handles the FUTURE response asynchronously
    "This actor test" should "this return a message" in {
      val testDuration = Duration(2, SECONDS)
      implicit val timeout = Timeout(testDuration)

      val myActorRef = system.actorOf(Props[MyActor])

      val events = Await.result(eventActorRef ? 
                     EventsListQuery("jboner","akka"),
                       testDuration).asInstanceOf[List[MyReturnData]]

      events should not be ('empty)
    }

!SLIDE transition=blindY
# References

* [Mockito](http://code.google.com/p/mockito/)
* [Specs2 User Guide](http://etorreborre.github.com/specs2/guide/org.specs2.UserGuide.html#User+Guide)
* [Testing Actor Systems (Scala)](http://doc.akka.io/docs/akka/2.0/scala/testing.html#integration-testing-with-testkit)
* [Maciej Matyjas - Testing Blocking Messages with Akka's TestKit](http://maciejmatyjas.com/2012/02/23/testing-blocking-messages-with-akkas-testkit/)