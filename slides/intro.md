
```java
interface Framework { 
  <T> Class<? extends T> secure(Class<T> type); 
}

@interface Secured { 
  String user(); 
} 

class UserHolder { 
  static String user = "ANONYMOUS"; 
}
```

```java
class Service { 
  @Secured(user = "ADMIN") 
  void deleteEverything() { 
    // delete everything... 
  } 
} 
```

----
## Class redefinition
(build time, agent)

```java
class Service { 
  @Secured(user = "ADMIN") 
  void deleteEverything() { 
    // delete everything... 
  } 
} 
```


```java
class Service {
  @Secured(user = "ADMIN")
  void deleteEverything() {
    if(!"ADMIN".equals(UserHolder.user)) { 
      throw new IllegalStateException("Wrong user"); 
    } 
    // delete everything... 
  } 
}
```

----
## Create subclass
(Liskov substitution)

```java
class Service { 
  @Secured(user = "ADMIN") 
  void deleteEverything() { 
    // delete everything... 
  } 
} 
```


```java
class SecuredService extends Service { 
  @Override 
  void deleteEverything() {
    if(!"ADMIN".equals(UserHolder.user)) { 
      throw new IllegalStateException("Wrong user"); 
    } 
    super.deleteEverything(); 
  } 
} 
```

----

## The “black magic” prejudice

```js 
var service = { 
  /* @Secured(user = "ADMIN") */
  deleteEverything: function () { 
    // delete everything ... 
  }
}
```

```js 
function run(service) { 
 //No type, no problem. (“duck typing”)
  service.deleteEverything(); 
}
```

- In dynamic languages (also those running on the JVM) this concept is applied a lot!
- For framework implementors, type-safety is conceptually impossible but we want it. 
- When type information available, we are able to fail fast when generating 
code in case that types don't match.

----
## The “black magic” prejudice
<div style="position:absolute; left:0px;  width:1000px;font-size: 18pt;"  align="left">
<p>The Java language comes with a comparatively strict type system. Java requires all variables and objects to be of a specific type and any attempt to assign incompatible types always causes an error. These errors are usually emitted by the Java compiler or at the very least by the Java runtime when casting a type illegally. Such strict typing is often desirable, for example when writing business applications. Business domains can usually be described in such an explicit manner where any domain item represents its own type. This way, we can use Java to build very readable and robust applications where mistakes are caught close to their source. Among other things, it is Java's type system that is responsible for Java's popularity in enterprise programming.
</p>
<p>
However, by enforcing its strict type system, Java imposes limitations that restrict the language's scope in other domains. For example, when writing a general-purpose library that is to be used by other Java applications, we are normally not able to reference any type that is defined in the user's application because these types are unknown to us when our library is compiled.
</p>
</div>

----

<img src="resources/Fig2.png" width="80%">



----
## Do-it-yourself as an alternative?

````java 
class Service {
  void deleteEverything() {
    if(!"ADMIN".equals(UserHolder.user)) { 
      throw new IllegalStateException("Wrong user"); 
    } 
    // delete everything... 
  } 
} 
```
- At best, this makes testing an issue.
  - Maybe still the easiest approach for simple cross-cutting concerns.
  - In general, declarative programming often results in readable and modular code.

----

## “Plain old Java applications” (POJAs)
> Working with POJOs reduces complexity. Reducing infrastructure code as a goal


---

<img src="resources/fig1.png" width="80%">


----
## Four solutions
- At design time
  - source to source transformation (Spoon)
  - postprocessing compilation using AspectJ or bytecode processor (ASM)
- At runtime
  - using reflection
  - using (byte)code generation at runtime and specific classloader or Java agent


---

# Source code analysis


----
## Astract Syntax Tree

An Abstract Syntax Tree is a data structure representing source code.

<img src="resources/Fig3.png" width="70%">

- Libraries exist to compute and manipulate ASTs.

----
## Astract Syntax Tree

- There can be many different ASTs for he same programming language
  - Typed vs untyped nodes
  - Use enum instead of classes
  - Different ways to traverse the AST
  - Read-only or read-write?

----
## Example: SrcML

SrcML provides XML-based ASTs that can be analyzed with any DOM technology

```XML
<unit>
<comment type="line">// copy the input to the output</comment>
<while>while 
  <condition>(<expr><name><name>std</name>::<name>cin</name></name> &gt;&gt; <name>n</name></expr>)</condition>
  <expr_stmt><expr><name><name>std</name>::<name>cout</name></name> &lt;&lt; <name>n</name> &lt;&lt; '\n'</expr>;</expr_stmt></while>
</unit>
```
- **Pro**: toString is valid code
- **Pro**: Can be loaded in any XML ready library
- **Con**: no complete AST (very fine grain expressions are not handled)

----
## Tools for analyzing Astract Syntax Trees

- There are many tools for extracting and analyzing ASTs. They depend on the language to be parsed.
  - Spoon (Java)
  - SrcML (Java/C++)
  - JDT (Java)
  - AST (Python)
  - Clang (C)

----
## Example: Spoon
- Spoon provides Java ASTs:
  - That can be analyzed
  - That can be transformed
  - With a powerful API

----
## Navigating an AST

```bash 
java spoon.Launcher -g -i srcDir
```

```java 
public class Dojo {
  public int b;
  public double c;
  void foo(Object t, int i) {
     t.hashCode();
     b=i+3;
     b=i-5;
     b=i-98;
     b=b+i;
  }
}
```

<p class="fragment current-visible"
style="position:absolute; left:600px; top:150px;">
<img src="resources/Fig4.png" width="100%"/></p>

---
# Source Code Analysis

----
## AST analysis 
- AST analysis is useful is a number of different use cases:
  - For implementing Lint tools / bad smells detection
    - (your own, specialized Lint, FindBugs, PMD, Checkstyle, JLint)
  - Verification of contracts
    - Only use constructor through factory
    - e.g. no explicit exception reaches the main
  - Visualization 

----

## Basics of AST Analysis with Spoon
```java
// navigation
class.getFields()
field.getDefaultExpression()
if.getThenBranch();
method.getBody();
...
```

```java
// queries
list1 = methodBody.getElements(new TypeFilter(CtAssignment.class));

// collecting all deprecated classes
list2 = rootPackage.getElements(new AnnotationFilter(Deprecated.class));

// creating a custom filter to select all public fields
list3 = rootPackage.getElements(
  new AbstractFilter<CtField>(CtField.class) {
    public boolean matches(CtField field) {
      return field.getModifiers.contains(ModifierKind.PUBLIC);
    }});
```

----
## Spoon Analysis #1: Detecting empty catch blocks
```java
public class CatchProcessor extends AbstractProcessor<CtCatch> {

public void process(CtCatch element) {
  if (element.getBody().getStatements().size() == 0) {
    getFactory().getEnvironment().report(this, Severity.WARNING,
     element, "empty catch clause");
  }
}
}
```

```bash
java -cp spoon.jar spoon.Launcher -i sourceFolder -p CatchProcessor
```

----
## Spoon Analysis #2: Detecting public fields

```java
public class PublicFieldProcessorWarning extends AbstractProcessor<CtField<?>>{
  @Override
  public void process(CtField<?> arg0) {
    if (arg0.hasModifier(ModifierKind.PUBLIC)) {
      getEnvironment().report(this, Severity.WARNING, arg0, 
                                    "Found a public field");
    }
  }
}
````

---
# Source Code Transformation

----
## API for code transformation

A source code transformation tool provides you with an API.

<img src="resources/Fig5.png" width="100%"/>



<!--<div>
Scope  | Name | Description
-- | -- | -- 
generic {#nextsteps .emphasized} | *replace(element)* | replaces an element by another one.
generic | *insertBefore(element)* | "inserts the current element (""this"") before another element in a code block."
generic | *insertAfter(element)* | "inserts the current element (""this"") after another element in a code block."
block  | *insertBegin(element)*  | adds an element at the begin of a code block.
block  | *insertEnd(element)*  | appends an element at the end of a code block.
throw  | *setThrownExpression(e)* | sets the expression returning a throwable object.
assignment  | *setAssignment(e)* | sets the expression to be assigned in a variable.
if  | *setElseStatement(stmt)* | "sets the ""else"" statement of an if/then/else."
<div>-->

----
## Spoon Transformation #1: adding not-null checks

```java
public class NotNullCheckAdderProcessor extends
		AbstractProcessor<CtParameter<?>> {

  @Override
  public boolean isToBeProcessed(CtParameter<?> element) {
    return !element.getType().isPrimitive();// only for objects
  }

  public void process(CtParameter<?> element) {
    // we declare a new snippet of code to be inserted
     CtCodeSnippetStatement snippet = getFactory().Core().createCodeSnippetStatement();
    
    // this snippet contains an if check
    snippet.setValue("if(" + element.getSimpleName() + " == null "
          + ") throw new IllegalArgumentException(
                        \"[Spoon inserted check] null passed as parameter\");");
  
    // we insert the snippet at the beginning of the method boby
    element.getParent(CtMethod.class).getBody().insertBegin(snippet);
  }
	
}
```

----
## Spoon Transformation #2 (driven by annotations)

```java
public @interface Bound {
  double min(); 
}
....
public void openUserSpacePort(@Bound(min = 1025) int a) {
  // code to open a port
}
```

----
## Spoon Transformation #2 (driven by annotations)

```java
public class Bound2Processor extends
AbstractAnnotationProcessor<Bound, CtParameter<?>> {

public void process(Bound annotation, CtParameter<?> element) {
  // we declare a new snippet of code to be inserted
  CtCodeSnippetStatement snippet = getFactory().Core()
    .createCodeSnippetStatement();

  // this snippet contains an if check
  snippet.setValue("if("
    + element.getSimpleName() + " < " + annotation.min() + ")"
    +" throw new RuntimeException(\"[Spoon check] Bound violation\");");

  // we insert the snippet at the beginning of the method boby
  element.getParent(CtMethod.class).getBody().insertBegin(snippet);
} // end process
}
```

---
# Manipulating bytecode at runtime

----
## Existing Libraries
- ASM
- BCEL
- Javassist
- cglib
- ByteBudy

----
## ASM / BCEL
- Byte code-level API gives full freedom
- Requires knowledge of byte code (stack metaphor, JVM type system)
- Requires a lot of manual work (stack sizes / stack map frames)
- Byte code-level APIs are not type safe (jeopardy of verifier errors, visitor call order)
- Byte code itself is little expressive
- Low overhead (visitor APIs)
- ASM is currently more popular than BCEL (used by the OpenJDK, considered as public API)
- Versioning issues for ASM (especially v3 to v4)

----
## Javassist
- Old library
- Strings are not typed (“SQL quandary”)
- Specifically: Security problems!
- Makes debugging difficult(unlinked source code, exception stack traces)
- Bound to Java as a language
- The Javassist compiler lags behind javac
- Requires special Java source code instructions for realizing cross-cutting concerns


----
## cglib

```java 
class SecuredService extends Service { 
  @Override 
  void deleteEverything() { 
    methodInterceptor.intercept(this, 
      Service.class.getDeclaredMethod("deleteEverything"), 
      new Object[0], 
      new $MethodProxy()); 
  }
 
  class $MethodProxy implements MethodProxy { 
    // inner class semantics, can call super 
  } 
}

interface MethodInterceptor { 
  Object intercept(Object object, 
                   Method method, 
                   Object[] arguments, 
                   MethodProxy proxy) 
    throws Throwable 
}

```

----
## cglib

- Discards all available type information
- JIT compiler struggles with two-way-boxing(check out JIT-watch for evidence)
- Interface dependency of intercepted classes
- Delegation requires explicit class initialization(breaks build-time usage / class serialization)
- Subclass instrumentation only(breaks annotation APIs / class identity)
- “Feature complete” / little development
- Little intuitive user-API

----
## Byte Buddy
- code generation and manipulation library for creating and modifying Java classes during the runtime of a Java application and without the help of a compiler
- offers a convenient API for changing classes either manually, using a Java agent or during a build
- Built on top of ASM

----
## Byte Buddy
```java
Class<?> dynamicType = new ByteBuddy() 
  .subclass(Object.class) 
  .method(named("toString"))
  .intercept(value("Hello World!"))
  .make()   
  .load(getClass().getClassLoader(), 
        ClassLoadingStrategy.Default.WRAPPER) 
  .getLoaded(); 

....

assertThat(dynamicType.newInstance().toString(), 
           is("Hello World!"));

```

----
## Byte Buddy
```java
Class<?> dynamicType = new ByteBuddy() 
  .subclass(Object.class) 
  .method(named("toString"))
  .intercept(to(MyInterceptor.class))
  .make()   
  .load(getClass().getClassLoader(), 
        ClassLoadingStrategy.Default.WRAPPER) 
  .getLoaded();
```


----
## Byte Buddy
```java
Class<?> dynamicType = new ByteBuddy() 
  .subclass(Object.class) 
  .method(named("toString"))
  .intercept(to(MyInterceptor.class))
  .make()   
  .load(getClass().getClassLoader(), 
        ClassLoadingStrategy.Default.WRAPPER) 
  .getLoaded();
```

```java
class MyInterceptor { 
  static String intercept(@Origin Method m) { 
    return "Hello World from " + m.getName();
  } 
}
```

----
## Byte Buddy

```java
//Provides caller information
@Origin Method|Class<?>|String 

//Allows super method call
@SuperCall Runnable|Callable<?>

//Allows default method call
@DefaultCall Runnable|Callable<?>

//Provides boxed method arguments
@AllArguments T[]

//Provides argument at the given index
@Argument(index) T
```


----
## Byte Buddy

```java
//Provides caller instance
@This T

//Provides super method proxy
@Super T
```

----
## Byte Buddy

```java
class Foo { 
  String bar() { return "bar"; }
}

Foo foo = new Foo();

new ByteBuddy() 
  .redefine(Foo.class) 
  .method(named("bar"))
  .intercept(value("Hello World!"))
  .make()   
  .load(Foo.class.getClassLoader(), 
        ClassReloadingStrategy.installedAgent()); 

assertThat(foo.bar(), is("Hello World!"));
```
- The instrumentation API does not allow introduction of new methods.
- This might change with JEP-159: Enhanced Class Redefiniton.


----
## Byte Buddy

```java
class Foo { 
  String bar() { return "bar"; }
}

assertThat(new Foo().bar(), is("Hello World!"));
```


```java
public static void premain(String arguments, 
    Instrumentation instrumentation) {
  new AgentBuilder.Default()
    .rebase(named("Foo"))
    .transform( (builder, type) -> builder
           .method(named("bar"))
           .intercept(value("Hello World!")); 
     )
    .installOn(instrumentation);}
```

<p
style="position:absolute; left:800px; top:85px;">
<img src="resources/Fig6.png" width="100%"/></p>

<p 
style="position:absolute; left:800px; top:285px;">
<img src="resources/Fig6.png" width="100%"/></p>

----
## Byte Buddy
```java
class Foo { 
  @Qux 
  void baz(List<Bar> list) { }
}

Method dynamicMethod = new ByteBuddy() 
  .subclass(Foo.class) 
  .method(named("baz"))
  .intercept(StubMethod.INSTANCE)
  .attribute(new MethodAttributeAppender
                .ForInstrumentedMethod())
  .make()   
  .load(getClass().getClassLoader(), 
        ClassLoadingStrategy.Default.WRAPPER) 
  .getLoaded()
  .getDeclaredMethod("baz", List.class); 

assertThat(dynamicMethod.isAnnotatedWith(Qux.class), 
  is(true));
assertThat(dynamicMethod.getGenericParameterTypes()[0], 
  instanceOf(ParameterizedType.class));
```

---
## Comparison Spoon Vs ByteBuddy

- Spoon
  - Fined grained transformation with spoon
  - Works for GWT, Android
  - pleasant API

- ByteBuddy
  - allow to by pass the Java Type system
  - really efficient
  - pleasant API
  - can work with scala, ...
  - require Java agent to **redefine class**

  ----
  ## Next steps

- Write spoon processors, compile them to ASM transformations on bytecode or Byte Buddy
- Implement Model Type using Byte Buddy ?


