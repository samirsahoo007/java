# java
## How to set java path?
### Setting Temporary Path
`Open command prompt in Windows
Copy the path of jdk/bin directory where java located (C:\Program Files\Java\jdk_version\bin)
Write in the command prompt: SET PATH=C:\Program Files\Java\jdk_version\bin and hit enter command.
`

### Setting Permanent Path
`Go to My Computer ---> Right Click on it ---> Advanced System Settings ---> Advanced Tab ---> Click on Environment Variables
Click on New tab of User variables, assign value JAVA_HOME to Variable Name
java\jdk_version\bin path (copied path) to Variable Value and click on OK Button
Finally, click on OK button.`

# Memory Types in JVM
Understanding the different types of memory in JVM(Java Virtual Machine) is important for designing efficient and stable applications.

## 1. Heap Memory
When the JVM starts up, it creates the heap memory. This memory type represents a crucial component of the JVM as it stores all the objects created by the application.

The size of the memory may increase or decrease while the application runs. However, we can specify the initial size of the heap memory using the -**Xms** parameter:

`java -Xms4096M ClassName`

Furthermore, we can define the maximum heap size using the -**Xmx** parameter:

`java -Xms4096M -Xmx6144M ClassName`

If the application’s heap usage reaches the maximum size and it still requires more memory, it generates an **OutOfMemoryError**: Java heap space error.
The Java heap is allocated using **mmap**, or **shmat** if large page support is requested. 

## 2. Stack Memory
In this memory type, the JVM stores local variables and method information.

Furthermore, Java uses stack memory for thread execution. In an application, each thread has its own stack that stores information about the methods and variables it's currently using.

However, it's not managed by the Garbage Collection but by the JVM itself.

The stack memory has a fixed size, which is determined by the JVM at runtime. If the stack runs out of memory, the JVM throws the **StackOverflowError** error.

To avoid this potential problem, it’s essential to design the application to use the stack memory efficiently.

## 3. Native Memory (Off-heap memory)
The memory allocated outside of the Java heap and used by the JVM is called native memory. It is also known by the term off-heap memory.

Since the data in native memory is outside the JVM, we need to perform serialization to read and write data. Performance depends on the buffer, serialization process, and disk space.

Additionally, due to its placement outside the JVM, it’s not freed up by the Garbage Collector.

In native memory, the JVM stores thread stacks, internal data structures, and memory-mapped files.

The JVM and native libraries use native memory to perform actions that can’t be accomplished entirely in Java, such as interacting with the operating system or accessing hardware resources.

## 4. Direct Memory

Direct buffer memory is allocated outside the Java heap. It represents the operating system’s native memory used by the JVM process.

Java NIO uses this memory type to write data to the network or disc in a more efficient way.

Because direct buffers aren’t freed up by the Garbage Collector, their impact on the memory footprint of an application might not be obvious. Therefore, direct buffers should be allocated primarily to large buffers that are used in the I/O operations.

To use a direct buffer in Java, we call the allocateDirect() method on ByteBuffer:

ByteBuffer directBuf = ByteBuffer.allocateDirect(1024);

When loading files into memory, Java allocates a series of DirectByteBuffers using the direct memory. This way, it reduces the number of times the same bytes are copied. Buffers have a class responsible for freeing up the memory when the file is no longer needed.

We can limit the direct buffer memory size using the –XX:MaxDirectMemorySize parameter:

-XX:MaxDirectMemorySize=1024M

If native memory uses all the dedicated space for direct byte buffers, the OutOfMemoryError: Direct buffer memory error occurs.
