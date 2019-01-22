## Kotlin/Native 中的并发

  Kotlin/Native 运行时并不鼓励<!--
 -->带有互斥代码块与条件变量的经典线程式并发模型，因为已知该模型<!--
 -->易出错且不可靠。相反，我们建议使用一系列<!--
 -->替代方法，让你可以使用硬件并发并实现阻塞 IO。
 这些方法如下，并且分别会在后续各部分详细阐述：
   * 带有消息传递的 worker
   * 对象子图所有权转移
   * 对象子图冻结
   * 对象子图分离
   * 使用 C 语言全局变量的原始共享内存
   * 用于阻塞操作的协程（本文档未涉及）


### Worker

  Kotlin/Native 运行时提供了 worker 的概念来取代线程：并发执行的<!--
 -->控制流以及与其关联的请求队列。Worker 非常像参与者模型<!--
 -->中的参与者。一个 worker 可以与另一个 worker 交换 Kotlin 对象，从而在任何时刻<!--
 -->每个可变对象都隶属于单个 worker，不过所有权可以转移。
 请参见[对象转移与冻结](#transfer)部分。

  一旦以 `Worker.start` 函数调用启动了一个 worker，就可以使用其自身唯一的整数
 worker id 来寻址。其他 worker 或者非 worker 的并发原语（如 OS 线程）可以<!--
 -->使用 `execute` 调用向 worker 发消息。
 

  
 ```kotlin
val future = execute(TransferMode.SAFE, { SomeDataForWorker() }) {
   // 第二个函数参数所返回的数据
   // 作为“input”参数进入 worker 例程
   input ->
   // 这里我们创建了一个当有人消费结果 future 时返回的实例
   WorkerResult(input.stringParam + " result")
}

future.consume {
  // 这里我们查看从上文例程中返回的结果。请注意 future 对象或
  // id 都可以转移给另一个 worker，所以并不是必须要在
  // 获得 future 的同一上下文中消费之。
  result -> println("result is $result")
}
```



 The call to `execute` uses a function passed as its second parameter to produce an object subgraph
 (i.e. set of mutually referring objects) which is then passed as a whole to that worker, it is then no longer
 available to the thread that initiated the request. This property is checked if the first parameter
 is `TransferMode.SAFE` by graph traversal and is just assumed to be true, if it is `TransferMode.UNSAFE`.
 The last parameter to `execute` is a special Kotlin lambda, which is not allowed to capture any state,
 and is actually invoked in the target worker's context. Once processed, the result is transferred to whatever consumes
 it in the future, and it is attached to the object graph of that worker/thread.

  If an object is transferred in `UNSAFE` mode and is still accessible from multiple concurrent executors,
 program will likely crash unexpectedly, so consider that last resort in optimizing, not a general purpose
 mechanism.

  For a more complete example please refer to the [workers example](https://github.com/JetBrains/kotlin-native/tree/master/samples/workers)
 in the Kotlin/Native repository.

<a name="transfer"></a>
### Object transfer and freezing

   An important invariant that Kotlin/Native runtime maintains is that the object is either owned by a single
  thread/worker, or it is immutable (_shared XOR mutable_). This ensures that the same data has a single mutator, and so there is no need for locking to exist. To achieve such an invariant, we use the concept of not externally referred object subgraphs.
  This is a subgraph which has no external references from outside of the subgraph, which could be checked
  algorithmically with O(N) complexity (in ARC systems), where N is the number of elements in such a subgraph.
  Such subgraphs are usually produced as a result of a lambda expression, for example some builder, and may not
  contain objects, referred to externally.

   Freezing is a runtime operation making a given object subgraph immutable, by modifying the object header
  so that future mutation attempts throw an `InvalidMutabilityException`. It is deep, so
  if an object has a pointer to other objects - transitive closure of such objects will be frozen.
  Freezing is a one way transformation, frozen objects cannot be unfrozen. Frozen objects have a nice
  property that due to their immutability, they can be freely shared between multiple workers/threads
  without breaking the "mutable XOR shared" invariant.

   If an object is frozen it can be checked with an extension property `isFrozen`, and if it is, object sharing
 is allowed. Currently, Kotlin/Native runtime only freezes the enum objects after creation, although additional
 autofreezing of certain provably immutable objects could be implemented in the future.

<a name="detach"></a>
### Object subgraph detachment

   An object subgraph without external references can be disconnected using `DetachedObjectGraph<T>` to
  a `COpaquePointer` value, which could be stored in `void*` data, so the disconnected object subgraphs
  can be stored in a C data structure, and later attached back with `DetachedObjectGraph<T>.attach()` in an arbitrary thread
  or a worker. Combining it with [raw memory sharing](#shared) it allows side channel object transfer between
  concurrent threads, if the worker mechanisms are insufficient for a particular task.


<a name="shared"></a>
### Raw shared memory

  Considering the strong ties between Kotlin/Native and C via interoperability, in conjunction with the other mechanisms
 mentioned above it is possible to build popular data structures, like concurrent hashmap or shared cache with
 Kotlin/Native. It is possible to rely upon shared C data, and store in it references to detached object subgraphs.
 Consider the following .def file:
 


```c
package = global

---
typedef struct {
  int version;
  void* kotlinObject;
} SharedData;

SharedData sharedData;
```



After running the cinterop tool it can share Kotlin data in a versionized global structure,
and interact with it from Kotlin transparently via autogenerated Kotlin like this:



```kotlin
class SharedData(rawPtr: NativePtr) : CStructVar(rawPtr) {
    var version: Int
    var kotlinObject: COpaquePointer?
}
```



So in combination with the top level variable declared above, it can allow looking at the same memory from different
threads and building traditional concurrent structures with platform-specific synchronization primitives.

<a name="top_level"></a>
### Global variables and singletons

  Frequently, global variables are a source of unintended concurrency issues, so _Kotlin/Native_ implements
the following mechanisms to prevent the unintended sharing of state via global objects:

   * global variables, unless specially marked, can be only accessed from the main thread (that is, the thread
   _Kotlin/Native_ runtime was first initialized), if other thread access such a global, `IncorrectDereferenceException` is thrown
   * for global variables marked with the `@kotlin.native.ThreadLocal` annotation each threads keeps thread-local copy,
   so changes are not visible between threads
   * for global variables marked with the `@kotlin.native.SharedImmutable` annotation value is shared, but frozen
   before publishing, so each threads sees the same value
   * singleton objects unless marked with `@kotlin.native.ThreadLocal` are frozen and shared, lazy values allowed,
   unless cyclic frozen structures were attempted to be created
   * enums are always frozen

 Combined, these mechanisms allow natural race-freeze programming with code reuse across platforms in MPP projects.
