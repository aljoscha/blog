## On Structuring and Maintaining a Large Code Base (Apache Flink)

For me, an important principle in software engineering is separation of
concerns.  Code should be split into several modules that ideally have few
interdependencies. For a data processing framework, such as Flink, this is most
important when it comes to separating the SDK/API from the runtime. The SDK is
the part that users use to specify data processing pipelines while the runtime
is the part that executes those pipelines.

When doing small-scale projects or getting started on bigger ones it is easy to
underestimate the impact of good module structure on the long-term
maintainability and growth of the project. I want to use the example of Apache
Flink, a code base I'm familiar with, to show where we got the structure right,
where we got it wrong, and what effects this had (and has). I'm hoping this
will help you avoid similar pitfalls or maybe realize that you are in a similar
situation.

Spoiler alert, not all is lost for Flink's module structure and in the last
section I will sketch a plan for fixing the module structures that we got
wrong.

### What is Apache Flink?

[Apache Flink](https://flink.apache.org) is an open-source distributed data
processing framework. It consists of an SDK with APIs for expressing
[dataflow](https://en.wikipedia.org/wiki/Dataflow) programs and a distributed
runtime for executing said programs on a high number of machines in parallel.
There are different APIs depending on the use case: the [DataStream
API](https://ci.apache.org/projects/flink/flink-docs-master/dev/datastream_api.html)
for unbounded programs, the [DataSet
API](https://ci.apache.org/projects/flink/flink-docs-master/dev/batch/) for
bounded programs and the [Table
API/SQL](https://ci.apache.org/projects/flink/flink-docs-master/dev/table/) for
relational/analytical programs.  Please check out the
[documentation](https://ci.apache.org/projects/flink/flink-docs-master/) for a
more thorough introduction than I can give here. The [intro to the DataStream
API](https://ci.apache.org/projects/flink/flink-docs-master/learn-flink/datastream_api.html)
is a good place to get a feel for the API and for what the system does.

This is a short example that shows the important parts and concepts of a Flink
program (this uses the DataStream API but the DataSet API looks very similar):

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime);


DataStream<String> input = env.fromElements("hello, this is a sentence", "and this is another one");

input
    .flatMap(new FlatMapFunction<String, WordWithCount>() {
      @Override
      public void flatMap(String value, Collector<WordWithCount> out) {
        for (String word : value.split("\\s")) {
          out.collect(new WordWithCount(word, 1L));
        }
      }
    })
    .keyBy(WordWithCount::getWord)
    .timeWindow(Time.seconds(5))
    .reduce((a, b) -> new WordWithCount(a.word, a.count + b.count))
    .print();

// now give it to the runtime for execution
env.execute();
```

As a user, you create an environment and then create one or many `DataStreams`
which represent collections or streams of data. On these `DataStreams` you can
apply transformations that yield new transformed streams, ad infinitum. At the
end you would sink that data into some external system but here we're just
printing it. In a real world use case you would read data from Kafka and write
it to Elasticsearch, for example.

An important concept to keep in mind is that these transformations don't
immediately work with data. Instead, a logical graph of transformations is
created that represents the complete user program. You will then give it to the
Flink runtime which will execute it in parallel on multiple machines, possibly
up to thousands of machines for the largest use cases.

Flink is a good case study because of the scale of the project. It is large in
multiple dimensions: in terms of code, the number of people that work on this
code, and the number of people that use it in critical applications. Looking at
the [code on GitHub](https://github.com/apache/flink/) at the time of writing,
I counted _193_ modules, a quick (and inaccurate) `cloc` revealed _1033615_ (~1
mio) lines of Java code across about _10000_ source files. The GitHub project
shows a bit more than 700 contributors. Good structure becomes more and more
important as the code itself grows and as the number of people working on that
code increases, and here we have both.

### Module structure of the DataSet API

I'll start by showing the structure of the DataSet API. This API is older than
the DataStream API but curiously enough we got the structure right on this one.
This will help understanding how the DataStream API is different below.

These are the modules that make up the functionality of the DataSet API, both
SDK and runtime parts:

 - `flink-core`: This is a common API/SPI module for different Flink APIs and
   runtime components. In this you will find user-defined function interfaces
   such as `MapFunction`, the `Configuration` class, `TypeInformation` (which
   is the basic interface of the Flink serialization stack), and other things.
   The module does not depend on any other Flink modules but almost all other
   modules depend on it. If you know [Apache Spark](https://spark.apache.org),
   this is not at all like the Spark `core` module which contains all the core
   API and runtime components of Spark.
 - `flink-java`: This module contains the user-facing parts of the DataSet API,
   such as the `DataSet` class, `ExecutionEnvironment`, etc. This does not
   contain any of the runtime parts, such as implementations for different
   operators.
 - `flink-runtime`: This module contains two things: 1) the low-level
   distributed runtime parts, network stack, `JobManager`, `TaskManager`,
   checkpointing code, and 2) the runtime operator implementations for the
   operations in the DataSet API.
 - `flink-optimizer`: This contains the glue code that can take a program
   written using the DataSet API and translate it to something that the runtime
   understands and can execute. This uses the runtime operator implementations
   from `flink-runtime`. The name optimizer comes from the fact that Flink has
   its origins in the data base community and this part was/is meant to
   translate the user specified program into an efficient runtime program.
 - `flink-clients`: Yet more glue code that deals with executing Flink programs
   on different cluster management frameworks such as YARN or Kubernetes. This
   also has the code for the CLI.

Let's look at the dependency structure of these modules. Both `flink-core` and
`flink-java` are user-facing parts of the SDK, `flink-runtime`,
`flink-optimizer`, and `flink-clients` contain the runtime code:

<img width="70%" height="70%" src="https://github.com/aljoscha/blog/blob/main/assets/original-sin/dataset-dependency-structure.png">

In this and the following figures that show dependency graphs a pointed arrow
_foo -> bar_ means that module _foo_ depends on module _bar_.

The neat thing about this is that the user facing parts only expose user facing
code. That is, the "surface" of the API or SDK is somewhat minimal and well
defined.

The reason why this dependency structure works is that the API is completely
decoupled from runtime implementation. The transformations that users apply on
DataSets will build a logical graph of transformations. When a user writes
`dataSet.map(...)` this creates a `MapOperator` which is a logical
representation of that transformation.

When it is time to execute a graph of `Operators`, Flink will first translate
to a `Plan`, then to an `OptimizedPlan`, and finally to a `JobGraph`. Different
optimizations or transformations can happen in the different stages:

<img width="70%" height="70%" src="https://github.com/aljoscha/blog/blob/main/assets/original-sin/dataset-translation.png">

The [appendix](#appendix-a-dataset-transformations) has slightly more details
on this process but they are not pertinent to what I'm trying to get across
here.

### Module structure of the DataStream API

These are the modules that make up the functionality of the DataStream API,
both SDK and runtime parts:

 - `flink-core`: (see [above](#module-structure-of-the-dataset-api))
 - `flink-streaming-java`: This module contains the user-facing parts of the
   DataStream API, such as the `DataStream` class,
   `StreamExecutionEnvironment`, etc. *In addition* this also contains the
   runtime operator implementations of the different transformations. This is
   the big difference to the DataSet API.
 - `flink-runtime`: (see [above](#module-structure-of-the-dataset-api)) The
   runtime operator implementations for the DataSet API are not used by the
   DataStream API, only the distributed runtime parts.
 - `flink-clients`: (see [above](#module-structure-of-the-dataset-api))

The dependency structure is simpler. Both `flink-core` and
`flink-streaming-java` contain the actual user-facing parts of the SDK while
`flink-runtime` and `flink-clients` are runtime modules:

<img width="70%" height="70%" src="https://github.com/aljoscha/blog/blob/main/assets/original-sin/datastream-dependency-structure.png">

The problem is that `flink-runtime` is a direct dependency of
`flink-streaming-java`. This means that users of our API/SDK will "see" all the
internal classes and can use them in their code. Additionally, our user-facing
API itself uses classes from `flink-runtime`, meaning users have to use those
internal classes. Why is that?

The Flink DataStream API doesn't have intermediate logical graph
representations that are decoupled from runtime operator implementations as the
DataSet API has. The runtime operator implementations reside in
`flink-streaming-java`, not `flink-runtime`. When a user calls `map()` on a
`DataStream`, this immediately instantiates the runtime operator for a map
operation. There are still some translation steps, but it works differently:

<img width="70%" height="70%" src="https://github.com/aljoscha/blog/blob/main/assets/original-sin/datastream-translation.png">

The `StreamGraph` is a legacy artifact and doesn't have any structure or
information that is different form the graph of `Transformations`. We have only
two steps: methods on `DataStream` immediately instantiate operator
implementations and then we have one step (disregarding `StreamGraph`) that
transforms this into a `JobGraph`. Here, we don't do any optimizations. The
only thing that happens is that operators can be chained together to allow them
to run in one Task.

### What problems do arise from this?

The root of the problem is that the runtime is "visible" to the users:
`flink-runtime` is a dependency of the SDK modules and some of the
interfaces/classes used in the SDK are actually internal classes in the
runtime. What this means in practice is that no-one can be on sure footing.
Users are using internal classes that might change from one release to the
next. Developers of Flink cannot change internal implementation details with
confidence because they might have seeped out into the public APIs and users
might be relying on them. This bleeding out of internal code to public APIs is
a subtle but steady process: as long as your SDK module depends on some runtime
modules it is easy to inadvertently expose internal code. You might start out
with a "clean" API but someone at some point will expose something internal.

One concrete example of internal code being part of the API is the
[`StreamOperator`](https://github.com/apache/flink/blob/f0ed29c06d331892a06ee9bddea4173d6300835d/flink-streaming-java/src/main/java/org/apache/flink/streaming/api/operators/StreamOperator.java)/[`AbstractStreamOperator`](https://github.com/apache/flink/blob/f0ed29c06d331892a06ee9bddea4173d6300835d/flink-streaming-java/src/main/java/org/apache/flink/streaming/api/operators/AbstractStreamOperator.java).
This is the basis of all runtime operator implementations behind the
DataStream transformations. However, there is also
[`DataStream.transform()`](https://github.com/apache/flink/blob/12a895aef63f17036d1b5234b6ebab1f0cb3e96d/flink-streaming-java/src/main/java/org/apache/flink/streaming/api/datastream/DataStream.java#L1214),
meaning users can directly write an operator and use it in their programs. If
you take a look at the imports on
[`AbstractStreamOperator`](https://github.com/apache/flink/blob/f0ed29c06d331892a06ee9bddea4173d6300835d/flink-streaming-java/src/main/java/org/apache/flink/streaming/api/operators/AbstractStreamOperator.java),
you will immediately see this uses a lot of classes from the runtime module.
This has access to the deepest details of
[`AbstractInvokable`](https://github.com/apache/flink/blob/f0ed29c06d331892a06ee9bddea4173d6300835d/flink-runtime/src/main/java/org/apache/flink/runtime/jobgraph/tasks/AbstractInvokable.java)
and
[`Environment`](https://github.com/apache/flink/blob/22112f12b07d20aed43705776cf93fbdc115ed23/flink-runtime/src/main/java/org/apache/flink/runtime/execution/Environment.java).
We wanted to give users the power to express everything that our internal
operators can do but we let the internals shine through and now cannot easily
change those internals without risking breakage in existing user code. Side
note: the
[`AbstractStreamOperator`](https://github.com/apache/flink/blob/f0ed29c06d331892a06ee9bddea4173d6300835d/flink-streaming-java/src/main/java/org/apache/flink/streaming/api/operators/AbstractStreamOperator.java)
is marked as
[`@PublicEvolving`](https://github.com/apache/flink/blob/dbcc456a652e980323b1b23692578e3c22e25e68/flink-annotations/src/main/java/org/apache/flink/annotation/PublicEvolving.java)
which tells users that they should expect breakage, in practice it's still
really bad to actually break code, though.

While the operator is an obvious example of internal code being exposed in the
API, there are more subtle things as well. For example,
[`CheckpointedFunction`](https://github.com/apache/flink/blob/55272b3a5652c16560a9f7391d1b50cb94410d7e/flink-streaming-java/src/main/java/org/apache/flink/streaming/api/checkpoint/CheckpointedFunction.java),
which is the interface to implement if you want to deal with state in
user-defined functions, is in the API module/package. Its methods, however,
have parameters that are in the runtime module/package, as can be seen from the
import statements.

The latest example of real-world breakage resulting from this entanglement of
the modules has come up during the release verification of the Flink 1.11.0
release. Take a look a [this
email](https://lists.apache.org/thread.html/re0e34736eea84ee88cd83b3bb51b4109fe5a31a55e5eb13fdb6174d9%40%3Cdev.flink.apache.org%3E).
Among other things, they use
[`CheckpointListener`](https://github.com/apache/flink/blob/f0ed29c06d331892a06ee9bddea4173d6300835d/flink-runtime/src/main/java/org/apache/flink/runtime/state/CheckpointListener.java),
which is defined in the runtime but is the interface to use if your user
functions need to interact with checkpointing notifications. Also, they use
[`ClientUtils`](https://github.com/apache/flink/blob/bac55c175b0d3a76395880d7aa2e5ae6484364fa/flink-clients/src/main/java/org/apache/flink/client/ClientUtils.java),
which is an internal class but has useful functionality for which no stable
equivalent exists in the API/SDK.

I think these issues point to a general problem that you get in large code
bases without clearly defined boundaries between the modules. If a project is
big enough you want to be able to have individual teams working on isolated
components and make progress on their own. If you don't clearly define the
interfaces between the modules you cannot isolate changes to modules. People
will step on each other's toes and you will see uncertainty about changes and
whether they might break user code. In the long run this decreases productivity
and the maintainability of the project.

### How can we get out of this bind in Flink?

This part might feel a bit too in-depth if you're not super familiar with Flink
or are a Flink developer, so feel free to skip to the conclusion.

I think there are roughly three things we need to do, in order:

 1. Provide user-facing, stable API for the functionality that
    `AbstractStreamOperator` provides.
 2. Turn the graph of `Transformations` into a logical graph and change the
    DataStream API to not directly instantiate runtime operators from the API.
 3. Reverse the dependency from `flink-streaming-java` -> `flink-runtime`, that
    is the SDK module must not depend on the runtime module.

We need to do _1._ because we need to get the operator out of the user-facing
API. In order to do that we must provide a stable alternative. There is a
reason why people write stream operators directly, which is that it provides
functionality that they otherwise don't get. If you can do everything the
operator can do by using a sanctioned API we can phase out the operator from the
API.

The graph of `Transformations` is currently a _physical_ graph, that is the
nodes directly represent runtime implementations of transformations. We need to
make this graph more similar to the DataSet API, where there are logical nodes
for the individual operations such as `map` or `filter`. Once we achieved that,
the representation of a user program in the API/SDK is decoupled from the
runtime. At execution time the streaming runtime would then take this logical
graph and turn it into a physical graph for execution.

Once we did the first to steps we can reverse the dependency structure. This
will probably require that we move some classes from the runtime package, such
as the `CheckpointListener` mentioned above.

These steps might seem easy but it will be a lengthy process. We cannot simply
break the existing API (which we do by removing the operator) but need to make
time for people to migrate to new abstractions once they are available. In the
long run, however, this better encapsulation or isolation of functionality in
modules will allow us to work more independently on the different modules. And
it will give the users more security when using sanctioned stable APIs.

Once we are done with everything, the module structure will probably look
something like this:

<img width="70%" height="70%" src="https://github.com/aljoscha/blog/blob/main/assets/original-sin/long-term-dependency-structure.png">

We will have `flink-core`, `flink-java`, and `flink-streaming` as the user-facing
parts of the SDK. They don't depend on any runtime modules, instead all the
runtime modules depend on the user-facing modules.

### Wrapping up

I hope that by using the real-world example of the Flink DataStream API I could
show you how important it is to think about module structure when you want to
develop a project that is maintainable over a long time. I also hope that we
can find the time to fix the problems in our dependency structure and come out
as a better project on the other side.

As a last thing, I want to mention our own Table API and the new [Flink
Stateful Functions](https://flink.apache.org/stateful-functions.html) projects
got the module structure right: this is the
[pom](https://github.com/apache/flink/blob/cd89167f685620895ea1dbebc187cbe2db93fa55/flink-table/flink-table-api-java/pom.xml)
of our Table API SDK module and this is the
[pom](https://github.com/apache/flink-statefun/blob/3f30aad0e22f03715cc5083bd573ccdcd2b8a6b2/statefun-sdk/pom.xml)
of the StateFun SDK module. It's good to see that we also do learn from past
mistakes.

## Appendix A: DataSet Transformations

The result of the `DataSet` operations is a graph of
[Operators](https://github.com/apache/flink/blob/4a5e8c083969cad215819ad6a3a0d2919e8c2d5e/flink-java/src/main/java/org/apache/flink/api/java/operators/Operator.java),
defined in `flink-java`. This gets translated to a
[Plan](https://github.com/apache/flink/blob/f0a99979b043875dca11b03b7aba9da6b5bd29bd/flink-core/src/main/java/org/apache/flink/api/common/Plan.java)
of
[Operators](https://github.com/apache/flink/blob/980d072fa2546dbc10cf878cf29532b2d8bbca8a/flink-core/src/main/java/org/apache/flink/api/common/operators/Operator.java)
(yes, same name, different class), defined in `flink-core`. This step is pretty
boring/mechanical, Flink instantiates some operators differently based on what
type of key is used but overall the resulting graph/plan is the same. The
optimizer then takes this, converts it to an
[OptimizedPlan](https://github.com/apache/flink/blob/59dd855628052c369b64c71edc1018ed378e8eec/flink-optimizer/src/main/java/org/apache/flink/optimizer/plan/OptimizedPlan.java)
which consists of
[PlanNodes](https://github.com/apache/flink/blob/9912de21a1053013a220707f8b3868bdbf93aaca/flink-optimizer/src/main/java/org/apache/flink/optimizer/plan/PlanNode.java),
defined in `flink-optimizer`. On this plan we then do optimizations, for
example picking the right join strategy (Nested-Loop Join, Hash Join, Merge
Join). The final step is translating this to a
[JobGraph](https://github.com/apache/flink/blob/f985ff36bc94acb3072356ae3250ef1a0a5a2f1f/flink-runtime/src/main/java/org/apache/flink/runtime/jobgraph/JobGraph.java),
this is where the actual
[Task](https://github.com/apache/flink/blob/98434249e63220baa77d5a6b16e2fad6515e163c/flink-runtime/src/main/java/org/apache/flink/runtime/taskmanager/Task.java)
implementations are wired together in a graph of
[JobVertices](https://github.com/apache/flink/blob/0114338da5ce52677d1dfa1ab4350b1567dc3522/flink-runtime/src/main/java/org/apache/flink/runtime/jobgraph/JobVertex.java).
Here, we also chain operations together to allow them to run in a single task.
A `Task` is the unit that the Flink runtime understands. They can read from
input and write to output and all the different operator implementations such
as Join/Reduce/Map are code that runs in a Task. The `JobGraph` defines the
connection patterns between the tasks. The code for this is in
`flink-optimizer` but the runtime operator implementations that it creates are
in `flink-runtime`, as well as the `JobGraph` and `JobVertex` which make up the
graph structure.










