
# Vlingo Reactive Platform

## Review and Get Going (a.k.a. Work Log)

1. Go to vlingo.io and navigate to the Getting Started and the [vlingo-hello example](https://docs.vlingo.io/getting-started/hello-world-1).
1. Install maven... Gosh, haven't used that in years.
1. Compiles after downloading the Internet, but tons of Warnings that multiple versions are found. Well... Not my job to dig into that right now.
    1. Instructions says that I can run `java -jar target/vlingo-helloworld-withdeps.jar 8080` but "-withdeps.jar" is not built.
    1. It would have been good to be a Maven Maverick to understand this. It is either a different "task" (or is it called "target") than `package` 
       that is needed.
    1. hmmm... On [StackOverflow](https://stackoverflow.com/questions/574594/how-can-i-create-an-executable-jar-with-dependencies-using-maven) 
       they say that one should execute `mvn clean compile assembly:single`, but that requires "assembly descriptor" which is not present.
    1. I am not going to be stopped by this.
1. Let's load up this project in Intellij IDEA.
    1. Ok, so there is an `assembly.xml`, and that is probably relevant. How do I "use" that? Do the Vlingo folks have this in the user-wide config? 
       I am so lost in Maven nonsense nowadays.
    1. Let's add a Assembly plugin into the POM and see if I can get that working.
    1. IntelliJ updating Maven indices takes a long time...
1. Compiling again with a Assembly plugin added to the POM. Maven downloads a new Internet ;-)... 6545 files in the brand new `$HOME/.m2/` 
   directory. Takes talent!
    1. `Descriptor with ID 'withdeps' not found`... Uhhh, that means that `assembly.xml` is not read.
    1. Ok, so there is a predefined `descriptorRef` called `jar-with-dependencies` which might work. There is a exclusion of `logback` 
       which could become problematic. And the name is different than what is in the documentation of `vlingo-helloworld`.
    1. Nah... I am dropping in the pointer to `assembly.xml` into the `pom.xml`
    
    ```xml
    <plugin>
        <artifactId>maven-assembly-plugin</artifactId>
        <version>3.3.0</version>
        <configuration>
          <descriptors>
            <descriptor>assembly.xml</descriptor>
          </descriptors>
        </configuration>
        <executions>
          <execution>
            <id>make-assembly</id> <!-- this is used for inheritance merges -->
            <phase>package</phase> <!-- bind to the packaging phase -->
            <goals>
              <goal>single</goal>
            </goals>
          </execution>
        </executions>
    </plugin>
    ```

1. Ok, I have the jar... with wrong name of course.
    1. No MainClass in Manifest... Grrr...
    1. F! Maven. Let's run inside IntelliJ instead. I suspect Vlingo people are inside Eclipse or something.
    1. IntelliJ says that there are all kinds of errors in the project structure that was built from the POM. That is almost always a bad indicator of POM
       and normally not a bug in IntelliJ. 
    1. Cleaning up, and the Vlingo helloworld starts with the help of `Boot.class`... But is that the right one?
    
1. Ok, so I am expected to use cURL to grab something... `curl -i -X GET http://localhost:18080/greetings/0`
    1. Nothing listening on that port. 10950? Empty reply.
    1. `XoomInitializer` is not called. I think I am starting with the wrong thing. I need to contact Vaughn...
    
1. Meanwhile, I got info that there is a TestCase for the failure inside a different branch. Let's check that out.
    1. Yeah, cool. A breaking testcase. And it is not even threaded, with consistent failure. This shouldn't be too difficult.
    1. The main problem troubleshooting this is that the lambda's are too anonymous. I suspect that there is a mistake in nested and next...
    1. Let's expand the lambdas and put in a `toString()` so we can differentiate.
    1. Well, that doesn't help much, since the lambdas are buried deep inside the CFCompletes data structure. Classic case of overengineering 
       so testing and troubleshooting suffers. Also an issue with Reactive programming in general, and not having state readily available is a problem.
       Let's see how we can get around that just for my own need now.
       
1. First I introduced a "name" used in CompletesId, so I could easier track which instance is what. And then after a lot of digging, I see that there is
   a FAILURE signalled on returning the `nested.andThen()`.
    1. Uhhh... maybe that is because I am single stepping and it timed out. Let's try this in full speed.
    1. Enabling the `debug()` stuff.
    1. Fixing up the text output in those debug messages...
    1. Yes! We have Failure...
    1. Why? Is it related to `FailureValue` vs return of a `Completes`??
    
1. So, what is `FailureValue`? It is populated down into the sequence of functions, and it seems that if any returns the failure value, then it is a failure.

1. IF SO, then why is the `nested` completes becoming a `failureValue`?
    1. Ah, so that happens because the `nested` is not completed yet.
    1. So, perhaps this failure signals `done` as well...
    1. Alright, so the output of this happens in a `if( future.isDone() )` block. Which future is that?
    1. After the `nested` is returning a `Completes` the rest of the chain in the `Completes<Integer> service` is called into the `hasFailed()` and 
       `isDone()` is `true`. This CAN NOT BE RIGHT.
       
1. Ok, I have found the problem. Since the lamdba that returns the `nested Completes`, the `future` of that outer `Completes` will be marked `Done`,
   since it contains a value. That in turn is not expected in the outer logic and things goes wrong. I don't think I should try to figure out
   how to fix this, until I have spoken to Vaughn. Whether it is a matter of a small patch, or whether to go for a larger refactoring.
   
1. The CFCompletes class is pretty badly designed, as for instance there are side-effects in query methods, which I think arise from the fact that a
   functional programming style is used, although states are maintained and changed within methods/functions. GutFeeling™ also indicates that this
   class is overly complex and possibly not thread-safe, although attempting to be thread-safe. In essence, if the single-thread case is barely 
   understandable, then it is extremely high probability that the multi-thread behavior is non-deterministic.



       
