FS2: Functional Streams for Scala (previously 'Scalaz-Stream')
=============

[![Build Status](https://travis-ci.org/functional-streams-for-scala/fs2.svg?branch=series/1.0)](http://travis-ci.org/functional-streams-for-scala/fs2)
[![Gitter Chat](https://badges.gitter.im/functional-streams-for-scala/fs2.svg)](https://gitter.im/functional-streams-for-scala/fs2)
[![Latest version](https://img.shields.io/maven-central/v/co.fs2/fs2-core_2.12.svg)](https://index.scala-lang.org/functional-streams-for-scala/fs2/fs2-core)

Quick links:

* [Microsite][microsite]
* [About the library](#about)
* [How to get latest version](#getit)
* API docs: [fs2-core][core-api], [fs2-io][io-api], [fs2-reactive-streams][rx-api], [fs2-experimental][experimental-api]
* [Docs and getting help](#docs)

[microsite]: http://fs2.io/index.html
[core-api]: https://oss.sonatype.org/service/local/repositories/releases/archive/co/fs2/fs2-core_2.12/1.0.0/fs2-core_2.12-1.0.0-javadoc.jar/!/fs2/index.html
[io-api]: https://oss.sonatype.org/service/local/repositories/releases/archive/co/fs2/fs2-io_2.12/1.0.0/fs2-io_2.12-1.0.0-javadoc.jar/!/fs2/io/index.html
[rx-api]: https://oss.sonatype.org/service/local/repositories/releases/archive/co/fs2/fs2-reactive-streams_2.12/1.0.0/fs2-reactive-streams_2.12-1.0.0-javadoc.jar/!/fs2/interop/reactivestreams/index.html
[experimental-api]: https://oss.sonatype.org/service/local/repositories/releases/archive/co/fs2/fs2-experimental_2.12/1.0.0/fs2-experimental_2.12-1.0.0-javadoc.jar/!/fs2/experimental/index.html

### <a id="about"></a>About the library ###

FS2 is a streaming I/O library. The design goals are compositionality, expressiveness, resource safety, and speed. Here's a simple example of its use:

```scala
import cats.effect.{ExitCode, IO, IOApp, Resource}
import cats.implicits._
import fs2.{io, text, Stream}
import java.nio.file.Paths
import java.util.concurrent.Executors
import scala.concurrent.ExecutionContext

object Converter extends IOApp {
  private val blockingExecutionContext =
    Resource.make(IO(ExecutionContext.fromExecutorService(Executors.newFixedThreadPool(2))))(ec => IO(ec.shutdown()))

  val converter: Stream[IO, Unit] = Stream.resource(blockingExecutionContext).flatMap { blockingEC =>
    def fahrenheitToCelsius(f: Double): Double =
      (f - 32.0) * (5.0/9.0)

    io.file.readAll[IO](Paths.get("testdata/fahrenheit.txt"), blockingEC, 4096)
      .through(text.utf8Decode)
      .through(text.lines)
      .filter(s => !s.trim.isEmpty && !s.startsWith("//"))
      .map(line => fahrenheitToCelsius(line.toDouble).toString)
      .intersperse("\n")
      .through(text.utf8Encode)
      .through(io.file.writeAll(Paths.get("testdata/celsius.txt"), blockingEC))
  }
  
  def run(args: List[String]): IO[ExitCode] =
    converter.compile.drain.as(ExitCode.Success)
}
```

This will construct a program that reads lines incrementally from `testdata/fahrenheit.txt`, skipping blank lines and commented lines. It then parses temperatures in degrees Fahrenheit, converts these to Celsius, UTF-8 encodes the output, and writes incrementally to `testdata/celsius.txt`, using constant memory. The input and output files will be closed upon normal termination or if exceptions occur.

Note that this example is specialised to `IO` for simplicity, but `Stream` is fully polymorphic in the effect type (the `F[_]` in `Stream[F, A]`), as long as `F[_]` is compatible with the `cats-effect` typeclasses.

The library supports a number of other interesting use cases:

* _Zipping and merging of streams:_ A streaming computation may read from multiple sources in a streaming fashion, zipping or merging their elements using an arbitrary function. In general, clients have a great deal of flexibility in what sort of topologies they can define, due to `Stream` being a first class entity with a very rich algebra of combinators.
* _Dynamic resource allocation:_ A streaming computation may allocate resources dynamically (for instance, reading a list of files to process from a stream built off a network socket), and the library will ensure these resources get released upon normal termination or if exceptions occur.
* _Nondeterministic and concurrent processing:_ A computation may read from multiple input streams simultaneously, using whichever result comes back first, and a pipeline of transformations can allow for nondeterminism and queueing at each stage. Due to several concurrency combinators and data structures, streams can be used as light-weight, declarative threads to build complex concurrent behaviour compositionally.

These features mean that FS2 goes beyond streaming I/O to offer a very general and declarative model for arbitrary control flow.

### <a id="docs"></a>Documentation and getting help ###

* [The official guide](https://functional-streams-for-scala.github.io/fs2/guide.html) is a good starting point for learning more about the library.
* [The FAQ](https://functional-streams-for-scala.github.io/fs2/faq.html) has frequently asked questions. Feel free to open issues or PRs with additions to the FAQ!
* Also feel free to come discuss and ask/answer questions in [the gitter channel](https://gitter.im/functional-streams-for-scala/fs2) and/or on StackOverflow using [the tag FS2](http://stackoverflow.com/tags/fs2). Gitter will generally get you a quicker answer.

Blog posts, talks, and other external resources are listed on the [Additional Resources](https://github.com/functional-streams-for-scala/fs2/wiki/Additional-Resources) page.

### <a id="getit"></a> Where to get the latest version ###

The latest version is 1.0.x. See the badge at the top of the README for the exact version number.

The [1.0 migration guide](https://github.com/functional-streams-for-scala/fs2/blob/series/1.0/docs/migration-guide-1.0.md)
summarizes the differences between 1.0 and 0.10. To get 1.0.x, add the following to your SBT build:

```
// available for Scala 2.11, 2.12
libraryDependencies += "co.fs2" %% "fs2-core" % "1.0.0" // For cats 1.4.0 and cats-effect 1.0

// optional I/O library
libraryDependencies += "co.fs2" %% "fs2-io" % "1.0.0"

// optional reactive streams interop
libraryDependencies += "co.fs2" %% "fs2-reactive-streams" % "1.0.0"

// optional experimental library
libraryDependencies += "co.fs2" %% "fs2-experimental" % "1.0.0"
```

The previous stable release is 0.10.6. You may want to first
[read the 0.10 migration guide](https://github.com/functional-streams-for-scala/fs2/blob/series/0.10/docs/migration-guide-0.10.md)
if you are upgrading from 0.9 or earlier. To get 0.10, add the following to your SBT build:

```
// available for Scala 2.11, 2.12
libraryDependencies += "co.fs2" %% "fs2-core" % "0.10.6"

// optional I/O library
libraryDependencies += "co.fs2" %% "fs2-io" % "0.10.6"
```

The fs2-core library is also supported on Scala.js:

```
libraryDependencies += "co.fs2" %%% "fs2-core" % "0.10.6"
```

API docs:

* [The core library][core-api]
* [The `io` library][io-api], FS2 bindings for NIO-based file I/O and TCP/UDP networking

### Projects using FS2 ###

If you have a project you'd like to include in this list, either open a PR or let us know in [the gitter channel](https://gitter.im/functional-streams-for-scala/fs2) and we'll add a link to it here.

* [circe-fs2](https://github.com/circe/circe-fs2): Streaming JSON manipulation with [circe](https://github.com/circe/circe).
* [doobie](https://github.com/tpolecat/doobie): Pure functional JDBC built on fs2.
* [fs2-aws](https://github.com/saksdirect/fs2-aws): FS2 streams to interact with AWS utilities
* [fs2-blobstore](https://github.com/lendup/fs2-blobstore): Minimal, idiomatic, stream-based Scala interface for key/value store implementations.
* [fs2-cassandra](https://github.com/Spinoco/fs2-cassandra): Cassandra bindings for fs2.
* [fs2-columns](https://gitlab.com/lJoublanc/fs2-columns): a `Chunk` that uses [shapeless](https://github.com/milessabin/shapeless) to store `case class` data column-wise.
* [fs2-cron](https://github.com/fthomas/fs2-cron): FS2 streams based on cron expressions.
* [fs2-crypto](https://github.com/Spinoco/fs2-crypto): TLS support for fs2.
* [fs2-elastic](https://github.com/amarrella/fs2-elastic): Simple client for Elasticsearch.
* [fs2-grpc](https://github.com/fiadliel/fs2-grpc): gRPC implementation for FS2 / Cats Effect.
* [fs2-http](https://github.com/Spinoco/fs2-http): Http server and client library implemented in fs2.
* [fs2-jms](https://github.com/kiambogo/fs2-jms): JMS connectors for FS2 streams
* [fs2-kafka](https://github.com/Spinoco/fs2-kafka): Simple client for Apache Kafka.
* [fs2-mail](https://github.com/Spinoco/fs2-mail): Fully asynchronous java non-blocking email client using fs2.
* [fs2-rabbit](https://github.com/gvolpe/fs2-rabbit): RabbitMQ stream-based client built on top of Fs2.
* [fs2-reactive-streams](https://github.com/zainab-ali/fs2-reactive-streams): A reactive streams implementation for fs2.
* [fs2-redis](https://github.com/gvolpe/fs2-redis): Redis stream-based client built on top of Fs2 / Cats Effect.
* [fs2-zk](https://github.com/Spinoco/fs2-zk): Simple Apache Zookeeper bindings for fs2.
* [http4s](http://http4s.org/): Minimal, idiomatic Scala interface for HTTP services using fs2.
* [mongosaur](https://gitlab.com/lJoublanc/mongosaur): fs2-based MongoDB driver.
* [scarctic](https://gitlab.com/lJoublanc/scarctic): fs2-based driver for [MAN/AHL's Arctic](https://github.com/manahl/arctic) data store.
* [scodec-protocols](https://github.com/scodec/scodec-protocols): A library for working with libpcap files. Contains many interesting pipes (e.g., working with time series and playing back streams at various rates).
* [scodec-stream](https://github.com/scodec/scodec-stream): A library for streaming binary decoding and encoding, built using fs2 and [scodec](https://github.com/scodec/scodec).
* [streamz](https://github.com/krasserm/streamz): A library that supports the conversion of [Akka Stream](http://doc.akka.io/docs/akka/2.4/scala/stream/index.html) `Source`s, `Flow`s and `Sink`s to and from FS2 `Stream`s, `Pipe`s and `Sink`s, respectively. It also supports the usage of [Apache Camel](http://camel.apache.org/) endpoints in FS2 `Stream`s and Akka Stream `Source`s, `Flow`s and `SubFlow`s.
* [upperbound](https://github.com/SystemFw/upperbound): A purely functional, interval-based rate limiter with support for backpressure.

### Related projects ###

FS2 has evolved from earlier work on streaming APIs in Scala and Haskell. Some influences:

* [Machines](https://github.com/ekmett/machines/), a Haskell library by Ed Kmett, which spawned [`scala-machines`](https://github.com/runarorama/scala-machines)
* [The FP in Scala stream processing library](https://github.com/fpinscala/fpinscala/blob/master/answers/src/main/scala/fpinscala/streamingio/StreamingIO.scala) developed for the book [FP in Scala](https://www.manning.com/books/functional-programming-in-scala)
* [Reflex](https://hackage.haskell.org/package/reflex), an FRP library in Haskell, by Ryan Trinkle
* There are various other iteratee-style libraries for doing compositional, streaming I/O in Scala, notably the [`scalaz/iteratee`](https://github.com/scalaz/scalaz/tree/scalaz-seven/iteratee) package and [iteratees in Play](https://www.playframework.com/documentation/2.0/Iteratees).

### Presentations, Blogs, etc. ###

See [Additional resources](https://github.com/functional-streams-for-scala/fs2/wiki/Additional-Resources).

### Acknowledgments ###

[![YourKit](https://www.yourkit.com/images/yklogo.png)](https://www.yourkit.com/)

Special thanks to [YourKit](https://www.yourkit.com/) for supporting this project's ongoing performance tuning efforts with licenses to their excellent product.

### Code of Conduct ###

See the [Code of Conduct](CODE_OF_CONDUCT.md).

