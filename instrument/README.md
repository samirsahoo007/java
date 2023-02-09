# Instrument

The instrumentation class is magic and doesn't have a constructor.

To obtain an instance, you must use the magic `Premain-Class` mechanism.

### Java Instrumentation API
Provides the ability to add byte-code to existing compiled Java classes.

## Setup
Throughout the article, we'll build an app using instrumentation.

Our application will consist of two modules:

1. An ATM app that allows us to withdraw money

2. And a Java agent that will allow us to measure the performance of our ATM by measuring the time invested spending money

The Java agent will modify the ATM byte-code allowing us to measure withdrawal time without having to modify the ATM app.

## Defining Java Agents

Java agents are part of the Java Instrumentation API. So to understand agents, we need to understand what instrumentation is.

Instrumentation, in the context of software, is a technique used to change an existing application, adding code to it. You can perform instrumentation both manually and automatically. You can also do it both at compiling time and runtime.

So, what is instrumentation good for?  It’s meant to allow you to change code, altering its behavior, without actually having to edit its source code file. This can be extremely powerful and also dangerous. What you can do with that is left to you. The possibilities are endless. Aspect-Oriented Programming? Mutation testing? Profiling? You name it.

With that out of the way, let’s focus again on Java agents. What are these things, and how do they relate to instrumentation?

In short, a Java agent is nothing more than a normal Java class. The difference is that it has to follow some specific conventions. The first convention has to do with the entry point for the agent. The entry point consists of a method called “premain,” with the following signature:

 ```public static void premain(String agentArgs, Instrumentation inst)```
 
If the agent class doesn’t have the “premain” method with the signature above, it should have the following, alternative method:

```public static void premain(String agentArgs) ```

As soon as the JVM initializes, it calls the premain method of every agent. After that, it calls the main method of the Java application as usual. Every premain method has to resume execution normally for the application to proceed to the startup phase.

The agent should have yet another method called “agentmain.” What follows are the two possible signatures for the method:
```
 public static void agentmain(String agentArgs, Instrumentation inst) 
 public static void agentmain(String agentArgs) 
```

Such methods are used when the agents are called not at JVM initialization, but after it.

## What Is a Java Agent
In general, a java agent is just a specially crafted jar file. It utilizes the Instrumentation API that the JVM provides to alter existing byte-code that is loaded in a JVM.

For an agent to work, we need to define two methods:

1. premain – will statically load the agent using -javaagent parameter at JVM startup

2. agentmain – will dynamically load the agent into the JVM using the Java Attach API

An interesting concept to keep in mind is that a JVM implementation, like Oracle, OpenJDK, and others, can provide a mechanism to start agents dynamically, but it is not a requirement.

## Loading a Java Agent

To be able to use the Java agent, we must first load it.

We have two types of load:

static – makes use of the premain to load the agent using -javaagent option

dynamic – makes use of the agentmain to load the agent into the JVM using the Java Attach API

### Static Load

Loading a Java agent at application startup is called static load. Static load modifies the byte-code at startup time before any code is executed.

Keep in mind that the static load uses the premain method, which will run before any application code runs, to get it running we can execute:

```java -javaagent:agent.jar -jar application.jar```

It's important to note that we should always put the –javaagent parameter before the –jar parameter.

Below are the logs for our command:
```
22:24:39.296 [main] INFO - [Agent] In premain method
22:24:39.300 [main] INFO - [Agent] Transforming class MyAtm
22:24:39.407 [main] INFO - [Application] Starting ATM application
22:24:41.409 [main] INFO - [Application] Successful Withdrawal of [7] units!
22:24:41.410 [main] INFO - [Application] Withdrawal operation completed in:2 seconds!
22:24:53.411 [main] INFO - [Application] Successful Withdrawal of [8] units!
22:24:53.411 [main] INFO - [Application] Withdrawal operation completed in:2 seconds!
```

We can see when the premain method ran and when MyAtm class was transformed. We also see the two ATM withdrawal transactions logs which contain the time it took each operation to complete.

Remember that in our original application we didn't have this time of completion for a transaction, it was added by our Java agent.

### Dynamic Load

The procedure of loading a Java agent into an already running JVM is called dynamic load. The agent is attached using the Java Attach API.

A more complex scenario is when we already have our ATM application running in production and we want to add the total time of transactions dynamically without downtime for our application.

Let's write a small piece of code to do just that and we'll call this class AgentLoader. For simplicity, we'll put this class in the application jar file. So our application jar file can both start our application, and attach our agent to the ATM application:

```
VirtualMachine jvm = VirtualMachine.attach(jvmPid);
jvm.loadAgent(agentFile.getAbsolutePath());
jvm.detach();
```

Now that we have our AgentLoader, we start our application making sure that in the ten-second pause between transactions, we'll attach our Java agent dynamically using the AgentLoader.

Let's also add the glue that will allow us to either start the application or load the agent.

We'll call this class Launcher and it will be our main jar file class:
```
public class Launcher {
    public static void main(String[] args) throws Exception {
        if(args[0].equals("StartMyAtmApplication")) {
            new MyAtmApplication().run(args);
        } else if(args[0].equals("LoadAgent")) {
            new AgentLoader().run(args);
        }
    }
}
```

```
Starting the Application
java -jar application.jar StartMyAtmApplication
22:44:21.154 [main] INFO - [Application] Starting ATM application
22:44:23.157 [main] INFO - [Application] Successful Withdrawal of [7] units!
```

#### Attaching Java Agent
After the first operation, we attach the java agent to our JVM:
```
java -jar application.jar LoadAgent
22:44:27.022 [main] INFO - Attaching to target JVM with PID: 6575
22:44:27.306 [main] INFO - Attached to target JVM and loaded Java agent successfully
```

Check Application Logs
Now that we attached our agent to the JVM we'll see that we have the total completion time for the second ATM withdrawal operation.

This means that we added our functionality on the fly, while our application was running:
```
22:44:27.229 [Attach Listener] INFO - [Agent] In agentmain method
22:44:27.230 [Attach Listener] INFO - [Agent] Transforming class MyAtm
22:44:33.157 [main] INFO - [Application] Successful Withdrawal of [8] units!
22:44:33.157 [main] INFO - [Application] Withdrawal operation completed in:2 seconds!
```

## Creating a Java Agent

After learning how to use an agent, let's see how we can create one. We'll look at how to use Javassist to change byte-code and we'll combine this with some instrumentation API methods.

Since a java agent makes use of the Java Instrumentation API, before getting too deep into creating our agent, let's see some of the most used methods in this API and a short description of what they do:

```
addTransformer – adds a transformer to the instrumentation engine
getAllLoadedClasses – returns an array of all classes currently loaded by the JVM
retransformClasses – facilitates the instrumentation of already loaded classes by adding byte-code
removeTransformer – unregisters the supplied transformer
redefineClasses – redefine the supplied set of classes using the supplied class files, meaning that the class will be fully replaced, not modified as with
retransformClasses
```

### Create the Premain and Agentmain Methods

We know that every Java agent needs at least one of the premain or agentmain methods. The latter is used for dynamic load, while the former is used to statically load a java agent into a JVM.

Let's define both of them in our agent so that we're able to load this agent both statically and dynamically:

```
public static void premain(
  String agentArgs, Instrumentation inst) {

    LOGGER.info("[Agent] In premain method");
    String className = "com.baeldung.instrumentation.application.MyAtm";
    transformClass(className,inst);
}
public static void agentmain(
  String agentArgs, Instrumentation inst) {

    LOGGER.info("[Agent] In agentmain method");
    String className = "com.baeldung.instrumentation.application.MyAtm";
    transformClass(className,inst);
}
```

In each method, we declare the class that we want to change and then dig down to transform that class using the transformClass method.

Below is the code for the transformClass method that we defined to help us transform MyAtm class.

In this method, we find the class we want to transform and using the transform method. Also, we add the transformer to the instrumentation engine:

```
private static void transformClass(
  String className, Instrumentation instrumentation) {
    Class<?> targetCls = null;
    ClassLoader targetClassLoader = null;
    // see if we can get the class using forName
    try {
        targetCls = Class.forName(className);
        targetClassLoader = targetCls.getClassLoader();
        transform(targetCls, targetClassLoader, instrumentation);
        return;
    } catch (Exception ex) {
        LOGGER.error("Class [{}] not found with Class.forName");
    }
    // otherwise iterate all loaded classes and find what we want
    for(Class<?> clazz: instrumentation.getAllLoadedClasses()) {
        if(clazz.getName().equals(className)) {
            targetCls = clazz;
            targetClassLoader = targetCls.getClassLoader();
            transform(targetCls, targetClassLoader, instrumentation);
            return;
        }
    }
    throw new RuntimeException(
      "Failed to find class [" + className + "]");
}

private static void transform(
  Class<?> clazz,
  ClassLoader classLoader,
  Instrumentation instrumentation) {
    AtmTransformer dt = new AtmTransformer(
      clazz.getName(), classLoader);
    instrumentation.addTransformer(dt, true);
    try {
        instrumentation.retransformClasses(clazz);
    } catch (Exception ex) {
        throw new RuntimeException(
          "Transform failed for: [" + clazz.getName() + "]", ex);
    }
}
```

With this out of the way, let's define the transformer for MyAtm class.

### Defining Our Transformer

A class transformer must implement ClassFileTransformer and implement the transform method.

We'll use Javassist to add byte-code to MyAtm class and add a log with the total ATW withdrawal transaction time:

```
public class AtmTransformer implements ClassFileTransformer {
    @Override
    public byte[] transform(
      ClassLoader loader,
      String className,
      Class<?> classBeingRedefined,
      ProtectionDomain protectionDomain,
      byte[] classfileBuffer) {
        byte[] byteCode = classfileBuffer;
        String finalTargetClassName = this.targetClassName
          .replaceAll("\\.", "/");
        if (!className.equals(finalTargetClassName)) {
            return byteCode;
        }

        if (className.equals(finalTargetClassName)
              && loader.equals(targetClassLoader)) {

            LOGGER.info("[Agent] Transforming class MyAtm");
            try {
                ClassPool cp = ClassPool.getDefault();
                CtClass cc = cp.get(targetClassName);
                CtMethod m = cc.getDeclaredMethod(
                  WITHDRAW_MONEY_METHOD);
                m.addLocalVariable(
                  "startTime", CtClass.longType);
                m.insertBefore(
                  "startTime = System.currentTimeMillis();");

                StringBuilder endBlock = new StringBuilder();

                m.addLocalVariable("endTime", CtClass.longType);
                m.addLocalVariable("opTime", CtClass.longType);
                endBlock.append(
                  "endTime = System.currentTimeMillis();");
                endBlock.append(
                  "opTime = (endTime-startTime)/1000;");

                endBlock.append(
                  "LOGGER.info(\"[Application] Withdrawal operation completed in:" +
                                "\" + opTime + \" seconds!\");");

                m.insertAfter(endBlock.toString());

                byteCode = cc.toBytecode();
                cc.detach();
            } catch (NotFoundException | CannotCompileException | IOException e) {
                LOGGER.error("Exception", e);
            }
        }
        return byteCode;
    }
}
```

### Creating an Agent Manifest File

Finally, in order to get a working Java agent, we'll need a manifest file with a couple of attributes.

Hence, we can find the full list of manifest attributes in the Instrumentation Package official documentation.

In the final Java agent jar file, we will add the following lines to the manifest file:

```
Agent-Class: com.baeldung.instrumentation.agent.MyInstrumentationAgent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Premain-Class: com.baeldung.instrumentation.agent.MyInstrumentationAgent
```

Our Java instrumentation agent is now complete. To run it, please refer to Loading a Java Agent section of this article.


