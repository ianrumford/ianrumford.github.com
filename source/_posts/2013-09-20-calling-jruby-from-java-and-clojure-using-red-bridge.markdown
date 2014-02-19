---
layout: post
title: "Calling  JRuby from Java and Clojure using Red Bridge"
date: 2013-09-20 18:00
comments: true
categories: 
---

> TL;DR: JRuby Red Bridge Embed Core is awesome!

A few months ago I needed to be able to call arbitrary methods in
arbitrary [JRuby][JRubyHome] scripts from an instance of a [Java][JavaHome] class.

Its turns out that [JRuby][JRubyWiki] has had a stable [API][RedBridgeAPIHome] called [**Red Bridge**][RedBridgeWikiHome] since version 1.4. 

This post show how to use **Red Bridge** from [Java][JavaHome], and also [Clojure][ClojureHome], to call arbitrary methods in arbitrary [JRuby][JRubyHome]  scripts.

The really nice thing about **Red Bridge** is that it will run "standard" Ruby without any fuss or bother e.g. you don't need to compile the Ruby to JVM bytecode.

It also means the Java / Clojure is decoupled from the Ruby (source);  you can change the Ruby very quickly and try again, making for a very quick development cycle.

<!-- more -->

# Some background on Red Bridge #

[**Red Bridge**][RedBridgeWikiHome] is the old name of the (Java) API
to [embed][RedBridgeWikiHome] [JRuby][JRubyWiki]. The new, more
correct name for the API is **JRuby Embed** but in this post and the
code I use **"Red Bridge"**.

From the [Red Bridge wiki][RedBridgeWikiHome]:

> Red Bridge consists of two layers: Embed Core on the bottom, and
implementations of JSR223 and BSF (Bean Scripting Framework) on top.

> Embed Core is JRuby-specific, and can take advantage of much of JRuby's power.

> JSR223 and BSF are more general interfaces that provide a common ground across scripting languages.

The [wiki][RedBridgeWikiHome] goes on to highlight the benefits of using **Embed Core**:

> With Embed Core, you can create several Ruby environments in one JVM, and configure them individually ...

> Embed Core offers several shortcuts, such as loading scripts from a java.io.InputStream, or returning Java-friendly objects from Ruby code. These allow you to skip a lot of boilerplate.

In the code below, I use the **Embed Core** [API][RedBridgeAPIHome], specifically the [ScriptingContainer][RedBridgeScriptingContainerAPIHome] class.

Its worth noting that the old **Red Bridge** [Project Kenai wiki][ProjectKenaiHome]  has some interesting examples not (seemingly) in the [new][RedBridgeWikiHome] wiki:  I used one from the Kenai [examples][ProjectKenaiRedBridgeExamples] page as the basis for my original code.

# Code is on Github #

The code can be found on [github][JRubyJavaClojureRedBridgeGithub].
Since I'm coming at this post principally from a
[Clojure][ClojureHome] perspective, its a [Leiningen][LeiningenHome]
project so you need Leiningen [installed][LeiningenGithub].

The project structure is Maven style e.g. _./src/main/java_,
_./src/main/clojure_ and _./src/main/ruby_.

Clone the repo in the usual way, get the dependencies, compile the Java, and set the classpath:

```bash
git clone git@github.com:ianrumford/jruby-java-clojure-red-bridge.git
cd ./jruby-java-clojure-red-bridge
lein deps
lein clean
lein javac
export CLASSPATH=`lein classpath`
```


# The JRuby Test Scripts #

The repo has two example JRuby scripts in _./src/main/ruby/bin_ (**java2jruby_example1.rb** and **clojure2jruby_example1.rb**) that will be  used  below.

These test scripts are very simple Ruby just to show the method call
and return mechanics. But it seems JRuby scripts called from **Red
Bridge** can do pretty much any Ruby they like, I have not found any
obvious or practical limitations.

# JRuby from Java #

The Java implementation can be found in the name.rumford.**JavaJRubyRedBridge** class.  The class contains the usual boilerplate of getters and setters, etc, familiar from, and to, all Java.

The below sections highlight the important  elements of the API.


## JRuby from Java: the Scripting Container ##

The **Scripting Container** can be thought of as the JRuby interpreter's _"handler"_ in the Java.  Its worth reading the [javadoc][RedBridgeScriptingContainerAPIHome] to get a feel for how to use the class, taking advantage of other capabilities.

Creating the container is easy, just an instantiation:

```java
  ScriptingContainer newContainer = new ScriptingContainer();
```


## JRuby from Java: the Scripting Receiver ##

The interface "contract" between the Ruby script and **Red Bridge** is very simple:  the Ruby script has to return something that will accept (receive) the method call(s).  This is called the **Scripting Receiver** in my code.

To me, the simplest way to  provide a receiver is to return an instance of a JRuby class as the very last statement of the script.  For example:

```ruby
require 'java'

class MyHelloWorldClass
  def hello_world
    puts("Hello World!")
  end
end

MyHelloWorldClass.new

```

In the Java, the receiver is returned by calling the _runScriptlet_ method of the **Scripting Container**:

```java
   scriptReceiver = getScriptingContainer().runScriptlet(scriptReader, scriptPath);
```

In the above, the _scriptPath_ is the filesystem path to the Ruby script (e.g. _./src/bin/java2jruby_example1.rb_).

The _scriptReader_ provides the Java _Reader_" to read the script file:

```java
  Reader scriptReader = new BufferedReader(new FileReader(scriptPath));
```

## JRuby from Java: calling a Ruby method ##

Calling the Ruby method is straightforward but has a couple of wrinkles.

The **Scripting Container's** _callMethod_ method take a Java **Class** (_resultClass_ in the code below) to tell **Red Bridge** what the type of the returned value from JRuby (_resultValue_ below) will be.

For example, if a  HashMap is expected, _resultClass_ must be **HashMap.class**, **Boolean.class** for Ruby _true_ or _false_, etc.  To support different return types, a Java generic method has been used:

Note also, the signature of _callMethod_ is different depending on whether arguments (a Java array) for the Ruby method are being provided.

```java
  public <T> T callScriptingMethod(String methodName,
                                   Object[] methodArgs,
                                   Class<T> resultClass
                                   ) throws IOException {

    logger.info("JJRB::callScriptingMethod ==> >{}< METHOD >{}< ARGS >{}<", inspect(), methodName, methodArgs.length);

    T resultValue = null;

    if (methodArgs.length > 0) {
      resultValue = findScriptingContainer().callMethod(findScriptingReceiver(), 
                                                        methodName,
                                                        methodArgs,
                                                        resultClass
                                                        );

    } else {
      resultValue = findScriptingContainer().callMethod(findScriptingReceiver(), 
                                                        methodName,
                                                        resultClass
                                                        );
    }

    if (resultValue != null) {
      logger.info("JJRB::callScriptingMethod <== >{}< METHOD >{}< RESULT >{}< >{}<", inspect(), methodName, resultValue.getClass().getName(), resultValue);
    } else {
      logger.info("JJRB::callScriptingMethod <== >{}< METHOD >{}< RESULT IS NULL", inspect(), methodName);
    }

    return resultValue;

  }
```

Its worth observing (confirming) that the same method can be called
repeatedly using the same **Scripting Container / Receiver** pair.

Also, other methods (with different _resultValue_ types) can be called just by changing the _methodName_, _resultClass_ and _methodArgs_ (if any).

## JRuby from Java: running a trivial demo ##

Although designed to be used from other Java classes, **JavaJRubyRedBridge** has a small **main** method to illustrate a trivial demo that passes a HashMap to a Ruby method and expects one back:

```java
  public static void main(String[] args) throws IOException {

    /*

      A very simple test of passing a HashMap and expecting one one back

    */

    HashMap<String,String> methodArgs = new HashMap<String,String>();

    JavaJRubyRedBridge redBridge = new JavaJRubyRedBridge(args);

    methodArgs.put("method", redBridge.getRubyScriptMethod());
    methodArgs.put("path", redBridge.getRubyScriptPath());

    redBridge.callRubyMethod(redBridge.getRubyScriptMethod(),
                             methodArgs,
                             HashMap.class);

  }
```

The Ruby looks like this:

```ruby

# Java to Ruby Red Bridge Example 1

require "java";

import 'java.util.HashMap';

$DEBUG = true

class Java2JRubyExample1

  WHOAMI = :Java2JRubyExample1

  def method1(*args)
    eye = :method1

    $DEBUG && args.each_with_index {|a, i| puts("#{WHOAMI}::#{eye} arg >#{i}< >#{a.class}< >#{a}<") }

    rubyReturn = {"class" => self.class.name, "method" => eye.to_s}
    
    # java expects a HashMap
    returnValue = HashMap.new(rubyReturn)

    $DEBUG && puts("#{WHOAMI}::#{eye} returnValue >#{returnValue.class}< >#{returnValue}<")
    
    returnValue
    
  end
  

end

# return an instance of the class
Java2JRubyExample1.new
```

Run the demo like so:

```bash
java -cp `lein classpath` name.rumford.JavaJRubyRedBridge --path ./src/main/ruby/bin/java2jruby_example1.rb --method method1
```

You should see some debug output from both Java and Ruby:

```bash
17:27:32.491 [main] INFO  name.rumford.JavaJRubyRedBridge - JJRB::processArguments ==> UDF ARGS >4<
17:27:32.495 [main] INFO  name.rumford.JavaJRubyRedBridge - JJRB::processArguments ARG >--path<
17:27:32.495 [main] INFO  name.rumford.JavaJRubyRedBridge - JJRB::processArguments ARG >./src/main/ruby/bin/java2jruby_example1.rb<
17:27:32.495 [main] INFO  name.rumford.JavaJRubyRedBridge - JJRB::processArguments ARG >--method<
17:27:32.495 [main] INFO  name.rumford.JavaJRubyRedBridge - JJRB::processArguments ARG >method1<
17:27:32.514 [main] INFO  name.rumford.JavaJRubyRedBridge - JJRB::processArguments <== UDF ARGS >4<
17:27:32.514 [main] INFO  name.rumford.JavaJRubyRedBridge - JJRB::callRubyMethod ==>  METHOD >method1< DATA >1< 
17:27:32.514 [main] INFO  name.rumford.JavaJRubyRedBridge - JJRB::callScriptingMethod ==> METHOD >method1< ARGS >1<
17:27:32.593 [main] INFO  name.rumford.JavaJRubyRedBridge - JJRB::newScriptingContainer <=> NEW CONTAINER >org.jruby.embed.ScriptingContainer@6b776b55<
17:27:32.594 [main] INFO  name.rumford.JavaJRubyRedBridge - JJRB::newScriptingReceiver ==> SCRIPT PATH >./src/main/ruby/bin/java2jruby_example1.rb<
17:27:32.594 [main] INFO  name.rumford.JavaJRubyRedBridge - JJRB::newScriptingReader <=> SCRIPT PATH >./src/main/ruby/bin/java2jruby_example1.rb< READER >java.io.BufferedReader@6d27d091<
17:27:34.307 [main] INFO  name.rumford.JavaJRubyRedBridge - JJRB::newScriptingReceiver <== SCRIPT PATH >./src/main/ruby/bin/java2jruby_example1.rb< RECEIVER >#<Java2JRubyExample1:0x23d582a7><
Java2JRubyExample1::method1 arg >0< >Java::JavaUtil::HashMap< >{"path"=>"./src/main/ruby/bin/java2jruby_example1.rb", "method"=>"method1"}<
Java2JRubyExample1::method1 returnValue >Java::JavaUtil::HashMap< >{"class"=>"Java2JRubyExample1", "method"=>"method1"}<
17:27:34.314 [main] INFO  name.rumford.JavaJRubyRedBridge - JJRB::callScriptingMethod <== METHOD >method1< RESULT >java.util.HashMap< >{class=Java2JRubyExample1, method=method1}<
17:27:34.314 [main] INFO  name.rumford.JavaJRubyRedBridge - JJRB::callRubyMethod <==  METHOD >method1< RESULT >java.util.HashMap< >{class=Java2JRubyExample1, method=method1}<
```


# JRuby from Clojure #

I have separated out the "test" code (_./src/main/clojure/redbridge.blog1.clj_) from the namespace (file) that implements the **Red Bridge** interface (**redbridge.clojure-jruby**, _./src/main/clojure/redbridge.clojure_jruby.clj_).

The key functions to create (new) the **Red Bridge** interface are wrapped in convenience functions, i.e. _find-scripting-container_ and _find-scripting-receiver_ that take the **state atom** as the argument.

## JRuby from Clojure:  the Scripting Container and Receiver ##

Since the **Scripting Container / Receiver** pair can be used for repeated calls, they are kept in a **state atom** holding  a map initialised with the values of the Ruby script path and method (if supplied):

```clojure
(defn initialize-scripting-state
  "create a starter state"
  ([ruby-script] (initialize-scripting-state ruby-script nil))
  ([ruby-script ruby-method]
     (atom {:script ruby-script :method ruby-method})))
```

Creating a Java _Reader_ for the Ruby script is simple:

```clojure
(defn new-script-reader
  "create a Java Reader pointing at the ruby script"
  [ruby-script]
  (let [file-reader (FileReader. ruby-script)
        buffered-reader (BufferedReader. file-reader)]
    buffered-reader))

```

As is creating a simple **Scripting Container** instance:

```clojure
(defn new-scripting-container
  "create a new jruby scripting container"
  []
  (org.jruby.embed.ScriptingContainer.))
```

And the **Scripting Receiver** is not much more involved:

```clojure
(defn new-scripting-receiver
  "parse a ruby script and return a receiver"
  ([ruby-script] (new-scripting-receiver ruby-script (new-scripting-container)))
  ([ruby-script scripting-container]
     (let [scripting-receiver (.runScriptlet scripting-container (new-script-reader ruby-script) ruby-script)]
       scripting-receiver)))
```

## JRuby from Clojure: calling a Ruby method ##

The function to call a Ruby method takes the **state atom**, the return value (note e.g just **HashMap** not **HashMap.class**), the method name and method arguments (if any).

if a method name is not supplied, it is fetched from the state atom.

The method's arguments, if any, are converted into a Java object array (_object-array_ function).

```clojure
(defn call-ruby-method
  "call the ruby method"
  [scripting-state result-class method-name & method-args]
  (let [scripting-container (find-scripting-container scripting-state)
        scripting-receiver (find-scripting-receiver scripting-state)
        scripting-name (if-not (nil? method-name)
                         method-name
                         (:method @scripting-state))]
    (if (not-empty method-args)
      (.callMethod scripting-container scripting-receiver scripting-name (object-array method-args) result-class)
      (.callMethod scripting-container scripting-receiver scripting-name result-class))))
```
      
## JRuby from Clojure: running a trivial command line demo ##

The Clojure "test" code (_redbridge.blog1_) has a  **-main** method that will run a simple demo in the same vein as the Java:

```clojure
(defn -main
  "main method to run a simple demo"
  [& args]
  (cjrb/call-ruby-method (cjrb/initialize-scripting-state "./src/main/ruby/bin/clojure2jruby_example1.rb") RubyHash "method1" args))
```

Run the demo using Leiningen:

```bash
lein run -m redbridge.blog1 "Hello" "World"
```

You should see the debug output from the Ruby:

```ruby
Clojure2JRubyExample1::method1 arg >0< >Java::ClojureLang::ArraySeq< >("Hello" "World")<
Clojure2JRubyExample1::method1 returnValue >Hash< >{:class=>"Clojure2JRubyExample1", "method"=>"method1"}<
```

## JRuby from Clojure: running a REPL demo ##

Trying stuff out in Clojure is much more fun and productive with the REPL.

These examples are in _redbridge_blog1_ which uses the _clojure2ruby.rb_ file, where _method1_ looks like this:

```ruby
  def method1(*args)
    eye = :method1

    $DEBUG && args.each_with_index {|a, i| puts("#{WHOAMI}::#{eye} arg >#{i}< >#{a.class}< >#{a}<") }

    # return value must be a Ruby Hash
    returnValue = {"class" => self.class.name, "method" => eye.to_s}

    $DEBUG && puts("#{WHOAMI}::#{eye} returnValue >#{returnValue.class}< >#{returnValue}<")
    
    returnValue    
    
  end
```

Do some initialisation:

```clojure
;; Clojure2JRubyExample1
;; *********************

;; Ruby script path
(def ruby-script-clojure2jruby-example1 "./src/main/ruby/bin/clojure2jruby_example1.rb" )

;; Initialize the state atom
(def state-clojure2jruby-example1 (cjrb/initialize-scripting-state ruby-script-clojure2jruby-example1))
```

Run same example as the command line demo:

```clojure
;; method1: no arguments, returns a Ruby Hash
;; *******
(def c2j-m1 (cjrb/call-ruby-method state-clojure2jruby-example1 RubyHash "method1"))
;; =>
{"class" "Clojure2JRubyExample1", "method" "method1"}
```

Although a Ruby Hash, you can do stuff with it in Clojure:

```clojure
;; Can iterate over the returned Ruby Hash
(for [[k v] c2j-m1] (lgr/info (format "k >%s< >%s< v >%s< >%s<" (class k) k (class v) v)))
;; => Logging on the REPL console
```

On the REPL console you should see:

```bash
16:19:25.156 [nREPL-worker-25] INFO  redbridge.blog1 - k >class java.lang.String< >class< v >class java.lang.String< >Clojure2JRubyExample1<
16:19:25.159 [nREPL-worker-25] INFO  redbridge.blog1 - k >class java.lang.String< >method< v >class java.lang.String< >method1<
```

And common map functions work just fine:

```clojure
;; common map functions

(keys c2j-m1)
;; => 
'("class" "method")

(vals c2j-m1)
;; =>
'("Clojure2JRubyExample1" "method1")

(count c2j-m1)
;; =>
2

(into {} (map (fn [[k v]] [(string/capitalize k) (string/upper-case v)]) c2j-m1))
;; =>
{"Class" "CLOJURE2JRUBYEXAMPLE1", "Method" "METHOD1"}
```

There are a few other example Ruby methods in _clojure2jruby.rb_ but you get the idea from the above I'm sure: just change the return type, method name and method arguments (if any) in the _call-ruby-method_ function call.

# Final Words #

I've only scratched the surface of **Red Bridge Embed Call** but its
an excellent interface between Java and Clojure and JRuby.

There are other ways of getting programs from different JVM
languages to talk to one another but this must be one of the most
robust and easiest to use, and possibly with the least performance penalty.  I've used **Red Bridge** to process larges volumes of data in a Hadoop pipeline and have not seen a failure due to **Red Bridge** or been aware of a bottleneck in the bridge itself.

An aside: the thing that's really struck home to me whilst preparing this
post is how _elegant_ the Clojure code is. I've been a long time fan
of Ruby and don't mind too much the _ceremony_ of Java. But just
scanning the three types of source code I've used here, my eye is
drawn to alight on the Clojure - its just looks so minimalist,
uncluttered and easy to understand.

BTW I was using Ubuntu 12.04, with Hotspot jdk 1.7.0_40, Clojure 1.5.1, JRuby 1.7.2, Leiningen 2.3.2, Emacs 24.3 and nREPL 0.1.5



[ClojureHome]: http:///clojure.org
[JRubyHome]: http://jruby.org/
[JavaHome]: http://www.java.com
[JRubyWiki]: https://github.com/jruby/jruby/wiki
[RedBridgeWikiHome]: https://github.com/jruby/jruby/wiki/RedBridge
[RedBridgeAPIHome]: http://jruby.org/apidocs/org/jruby/embed/package-summary.html
[RedBridgeScriptingContainerAPIHome]: http://jruby.org/apidocs/org/jruby/embed/ScriptingContainer.html
[ProjectKenaiHome]: https://kenai.com/projects/jruby/pages/RedBridge
[ProjectKenaiRedBridgeExamples]: https://kenai.com/projects/jruby/pages/RedBridgeExamples
[JRubyJavaClojureRedBridgeGithub]: https://github.com/ianrumford/jruby-java-clojure-red-bridge
[LeiningenHome]: http://leiningen.org/
[LeiningenGithub]: https://github.com/technomancy/leiningen
[Clojure Reducers Blog 1]: http://clojure.com/blog/2012/05/08/reducers-a-library-and-model-for-collection-processing.html
[Clojure Reducers Blog 2]: http://clojure.com/blog/2012/05/15/anatomy-of-reducer.html
[Clojure Reducers Code]: https://gist.github.com/ianrumford/6333358
