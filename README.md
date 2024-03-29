# Lab 2: MapReduce 

In this lab you will implement a few parts of a simplified but representative
MapReduce framework.

## First, some minutia

* Again, in this lab you will be using Git/GitHub and golang. There are
  references for both on the [course
  website](https://matthewlang.github.io/comp385).  [Git for
  CS](https://eagain.net/articles/git-for-computer-scientists/) is also a good
  reference for using Git.

* This lab will heavily reference the original [MapReduce
  paper](http://research.google.com/archive/mapreduce-osdi04.pdf), so being
  familiar with it and having it on hand as a reference will be important.

* You will also be using Google's
  [`glog`](https://godoc.org/github.com/golang/glog) package for leveled logging
  (rather than just using `fmt.Println` to give program output.
  
  The difference between logging and output is that output is meant to be
  consumed by the _user_ of the program, whereas logging is meant to be consumed
  by the _creator_ of the program. Logs are useful for telemetry and monitoring
  (especially for services, where logs for running services can be consumed),
  for autopsy purposes (as a way of seeing what happened leading up to an
  error), and for debugging when creating a program.
  
  Logging is _leveled_ when log entries can have different levels of severity
  for filtering and enabling.
  
  `glog` can be used to output logs at four main levels that are always enabled:

  ```
  glog.Infoln("This is generic 'This is happening now' type logs")
  glog.Warningln("This is for conditions that are not normal but might not be errors.")
  glog.Errorln("Uh-oh, something has gone wrong!")
  glog.Fatalln("Something has gone so wrong there's no point in continuing the program. Crash!")
  ```
  
  On top of these levels, there is also debug-level (aka _verbose_) logging.
  These are logs that would be too spammy to output when the program is running
  normally, but are nice to see when debugging a specific issue during
  development.

  ```
  glog.V(2).Infoln("RPC recieved: ", request.String())
  ```
  
  In this example, the verbosity level is 2; this log will only be logged when
  the program is run with the flag `-v=2` (or higher). Verbosity levels start
  from 1.
  
  Later, when you are touring my code in the lab, you will see verbose logging.
  This can be helpful when debugging. You should also use logging and _not_
  `printfln` for your implementations, as it is good practice.

  Make sure to take a look over the
  [`glog`](https://godoc.org/github.com/golang/glog) documentation.

* By default logs are written to disk in a temporary directory. If you want to
  see your logs when running your program, add the `-logtostderr` flag.

* I have set it up so that verbose logging is turned on in unit tests when the
  `-log.v` flag is set.

## Getting Started 

To get started, first clone your Github rep:
```
$ git clone https://github.com/[your github repo] mapreduce
```

The package `mapreduce/mapreduce` is the MapReduce (MR) library you will implement.

__As in the paper__, clients to the library write `Map` and `Reduce` functions,
create a MR `spec`, and call the `MapReduce` function.

This distributes their MR program over many worker processes and handles
scheduling and task management in a fault-tolerant way. Clients can also call
`LocalMapReduce` to run their MR locally.

## MapReduce

A MapReduce program that uses this lab's library will execute as follows:
1. A client program defines _map_ and _reduce_ functions, a set of input
   files they would like to use as input to the MapReduce, and a number of
   output reduce tasks.
1. A MR manager is started with this MR specification. It exposes a single RPC:
   `Register`. Workers call this RPC to register as able to work. To execute a
   phase (map or reduce), the manager distributes tasks to the workers that have
   registered and have no assigned work.
1. The manager considers each input file to be a single map task and has the
   worker run a `mapper` function (`mapreduce/mapper.go`) for each file/map task
   (either via RPC in the distributed case or by direct function call). The
   mapper function reads the its input file and calls the `MapFn` function with
   the file's contents. The mapper then writes its output to a set of files
   determined by the number of reducers.

   For each key, the mapper uses the hash of the key to determine which output
   file to write the values to.

   The files produced by the mapper have a specific format:
   `mrtmp.[jobname]-[mapid]-[reduceid]`. So, for a job named `foo` with 3
   mappers and 2 reducers, the map output will look like:

   ```
   mrtmp.foo-0-0
   mrtmp.foo-1-0
   mrtmp.foo-2-0
   mrtmp.foo-0-1
   mrtmp.foo-1-1
   mrtmp.foo-2-1
   ```

1. The manager has the workers execute the reduce phase by calling a `reducer`
   function (`mapreduce/reducer.go`) for each reduce task (again, either via RPC
   or direct function call). The reducer function should read its input and
   apply the job's `ReduceFn`.

   For reducer 1 in the above computation, the files it would read would be:

   ```
   mrtmp.foo-0-1
   mrtmp.foo-1-1
   mrtmp.foo-2-1
   ```

   The reduceFn's output should be written to the `reducer`'s specified output
   file.

   For reducer 1, this would be `mrtmp.foo-res-1`.

   For n reducers, there will be n output files:

   ```
   mrtmp.foo-res-0
   mrtmp.foo-res-1
   mrtmp.foo-res-2
   ...
   ```

1. After getting all map task outputs, the manager merges all the output files
   into a single file: `mrtmp.foo`. Note that this is unlike the paper, but
   convenient for the small-ish local data sets we will be testing on.
1. Finally, the manager cleans up after itself by shutting down workers and
   deleting any temporary files.

## Implementing MapReduce

The first thing that you should do in this lab is get familiar with the code for
the MR library in the `lab2/mapreduce` directory.

### Code tour

Here is the overall structure of the lab:

```
mapreduce
├── cleanup.sh ------------- script to clean up intermediate files.
├── data ------------------- test data for parts 2-4.
│   ├── ii-testout.txt
│   ├── wc-testout.txt
│   ├── pg-being_ernest.txt
│   └── ...
├── invertedindex ---------- part 4: inverted index.
│   └── main.go
├── mapreduce -------------- parts 1-4.
│   ├── api.go ------------- API for MapReduce components + helper functions.
│   ├── mapper.go ---------- part 1.1: Mapper.
│   ├── mapper_test.go ----- part 1.1: Mapper unit tests.
│   ├── mapreduce_test.go -- part 1: Overall unit tests.
│   ├── manager.go --------- MapReduce manager implementation.
│   ├── phase_executor.go -- parts 3-4: distributed + fault tolerant manager.
│   ├── reducer.go --------- part 1.2: Reducer.
│   ├── reducer_test.go ---- part 1.2: Reducer unit tests.
│   └── worker.go ---------- MapReduce worker implementation.
├── test-ii.sh ------------- part 5: script to test invertedindex.
├── test-ii-distributed.sh - part 5: script to test with a distributed manager.
├── test-wc.sh ------------- parts 2-3: script to test wordcount.
├── test-wc-distributed.sh - part 3: script to test with a distributed manager.
└── wordcount -------------- part 2: word count.
    └── main.go
```

The `mapreduce` folder contains the main code for the MR library, and will be
where you do the majority of your coding. The `wordcount` and `invertedindex`
folders will be used to implement MapReduces.

* `api.go` contains three main things:

    * Utility functions that your implementation will use.
    * The Request/Response structs for Go RPCs.
    * Implementation detail utility functions that won't be used by your
      program.

* `manager.go`: This file implements the MR manager. The `MapReduceSpec` is the
  spec (similar to the specification in the paper) of the MapReduce job to
  execute: it contains the job's name, the input files, the number of reducers
  (`R`), and the map and reduce functions.

  ```
  type MapReduceSpec struct {
    JobName  string                          // Name of the job.
    Files    []string                        // Input files.
    R        int                             // Number of reduce shards.
    MapFn    func(string, string) []KeyValue // Map function.
    ReduceFn func(string, []string) string   // Reduce function.
  }
  ```
  <center><i><small>The MapReduce spec</small></i></center>


  Notice that unlike the paper, the client does not specify the input
  splits/shards. In this lab, we will assume that _each map shard is exactly one
  input file_ (_i.e._, _M = `len(Files)`_).
  
  This means that we will have a map task for every input file.  
  In practice, this is close to most MapReduces, as input files are typically
  small, but very numerous.

  If you were to make this more sophisticated, you would create map tasks from
  evenly-sized shards of the input files, but that adds unnecessary complexity
  to this (already complex) project.

  In the manager there are a few important functions:

  * `LocalMapReduce` runs a sequential MR locally, given the spec. 
  * `MapReduce` runs a distributed MR. The `address` parameter is the address
    the manager process should listen on for worker registrations.
  * `Wait` waits for a running MR to complete and returns counters/stats about
    the MR to the caller.

  Both the `LocalMapReduce` and distributed `MapReduce` delegate execution of
  the map and reduce phases to functions named `mapper` and `reducer`, which are
  defined in `mapper.go` and `reducer.go`. We will discuss these functions in
  detail later, but they are the functions responsible for executing a single
  map/reduce task.

* `mapper.go` and `reducer.go` will be described in great detail in the next
  section. 
* `phase_executor.go` will be used later in the lab. The function `executePhase`
  is responsible for scheduling map and reduce tasks on distributed workers.
* `worker.go` is the code for the mapreduce worker. It details how the worker
  requests, get, commits, and fails to turn in work.

### Running a MapReduce

When executing a MapReduce, the following sequence of events occurs.

1. An application creates a `MapReduceSpec` that describes where the data for
   the MR input exists (before), or after (as in Class).
1. The application calls `LocalMapReduce` or `MapReduce` to start a MR. Each of
   these starts the mapreduce process and returns a pointer to a manager. The
   application will call `manager.Wait()` and wait for the MR to complete.

   ```
  
                                             ┌─────────┐
                                             │         │
   ┌──────────┐                              │ manager │
   │          │  launches                    │         │
   │   main   │ ──────────────►              └─────────┘
   │ program  │  new processes
   │          │                  ┌────────┐  ┌────────┐  ┌────────┐
   └──────────┘                  │        │  │        │  │        │
                                 │ worker │  │ worker │  │ worker │
                                 │        │  │        │  │        │
                                 └────────┘  └────────┘  └────────┘
   ```

   _Aside:_ For the purposes of this lab, we will manually run workers by
   starting different instances of the main program. Adding more complexity by
   including process management into our MapReduce implementation is unnecessary
   for the purpose of this lab.

1. The MR is run.  In the case of a local execution, the manager runs each stage
   sequentially by directly calling the `mapper` and `reducer` functions, which
   in turn execute the map and reduce functions specified by the user.

   The mapper functions write their output to reducer input shards, and, when
   the map phase is complete, the reducer functions read these inputs to
   construct their final output.

   In the distributed case, workers are started. Workers call a `Register` RPC
   on the manager to register themselves as available to work. The manager then
   runs the map phase by giving out available map tasks to available workers by
   calling the `DoWork` RPC on the workers. When the `DoWork` completes, the
   task has completed. When all map tasks have been completed, the manager then
   starts giving out reduce tasks to available workers.
1. When the reduce phase is done, the manager then merges all reducer output
   files to a single file (this is done for our lab only, where output files are
   small enough that we would want to have one view of the output).
1. The manager calls a `Shutdown` RPC on all workers to tell the workers to shut
   down.
1. `mapper.Wait()` returns to the application and returns counters from the run
   of the MR.

The main RPCs are `Register` (worker to manager), which makes a worker available
to do work; `DoWork` (manager to worker), which infoms a worker it should
perform a map/reduce task; and `Shutdown` (manager to worker), which informs the
worker that the MapReduce program has complete, and it may shut down.

```
 ┌────────┐                  ┌─────────┐
 │        │  Register        │         │
 │ worker ├─────────────────►│ manager │
 │        │                  │         │
 └────────┘                  └─────────┘


 ┌────────┐                  ┌─────────┐
 │        │         DoWork   │         │
 │ worker │◄─────────────────┤ manager │
 │        │                  │         │
 └────────┘                  └─────────┘


 ┌────────┐                  ┌─────────┐
 │        │        Shutdown  │         │
 │ worker │◄─────────────────┤ manager │
 │        │                  │         │
 └────────┘                  └─────────┘
```

## Part 1: Mapper/Reducer

For Part 1, your task is to implement the MapReduce worker. Specifically, the
parts of the worker that execute an individual map or reduce task.

At the completion of this part of the lab, you should be able to run all
`Sequential` unit tests. These tests simulate an entire MapReduce on
synthetically-generated input files and consist of simple map and reduce phases.
These tests are in `mapreduce/mapreduce_test.go`.

When you are _completely_ done with Part 1, make sure these tests pass.

```
go test mapreduce/mapreduce -run Sequential -test.v
```

### Part 1-1: Mapper

Your first step in the implementation is to write a function that runs a single
map task. 

The function `mapper` in `mapreduce/mapper.go` is a skeleton version of this;
your job is to complete it.

The purpose of this function is to read the input file to the map task from
disk, and call the `mapFn` function on its input. The output of the `mapFn` is a
list of key-value pairs. The `mapper` function needs to write these key-value
pairs to files that can be used for the reduce phase.

Please read the documentation on
[reading](https://gobyexample.com/reading-files) and
[writing](https://gobyexample.com/writing-files) files before beginning.

You can find functions for reading files and creating file writers in
[`io/ioutil`](https://golang.org/pkg/io/ioutil/).

In this lab, these files will be serialized using
[JSON](https://en.wikipedia.org/wiki/JSON). In the real world, these files would
be written in a more efficient binary format, but using JSON here means that you
can inspect the input and output files if you so desire.

The easiest way to write JSON-encoded output is with the JSON
[Encoder](https://golang.org/pkg/encoding/json/#Encoder) in the `encoding/json`
library in order to write JSON-encoded files. There is a little bit more
documentation
[here](https://blog.golang.org/json#streaming-encoders-and-decoders).

Notice that the `Encoder` is created with an `io.Writer` interface. Opening a
file for output in Go returns an implementation of the `io.Writer` interface for
that file. Similarly, when you'll read these files in the next section of the
lab, notice that the documentation for
[Decoder](https://golang.org/pkg/encoding/json/#Decoder) accepts an `io.Reader`
interface, which is the type returned from opening a file.

The `mapper` function has a set of unit tests that should pass when you are
complete.

```
go test mapreduce/mapreduce -run Mapper -test.v
```

Make sure that you read the comments in the source file, they are quite explicit
about how to implement this function.

### Part 1-2: Reducer

Now that your implementation can run map tasks, you must now implement the
ability to run reduce tasks.

`reducer` in `mapreduce/reducer.go` is the entry point for running reduce tasks.

The purpose of the reducer is to read all the input shards to the reduce task
and collect them into a single list of keys, each paired with a lists of values.

Then, each key and list of values should be provided to the `reduceFn` in sorted
order. The output of the `reduceFn` is a string. All outputs of the reducer
should then be written to the reducer's specified output file.

For this part, you should use the corresponding JSON
[Decoder](https://golang.org/pkg/encoding/json/#Decoder) to read the reducer's
input. The output of the reducer should also be encoded in JSON.

Run the `Reducer` unit tests to test your reducer.
```
go test mapreduce/mapreduce -run Reducer -test.v
```

If your implementations are correct, all tests for Part 1 should now pass.
```
go test mapreduce/mapreduce -run Sequential -test.v
```

Make sure that you commit and push your changes after finishing this part of the
lab.

## Part 2: WordCount

Now that you have a functioning sequential MapReduce implementation, let's write
a MapReduce!

The word count from the paper is the "hello world" of data processing and
appears as an example in the
[documentation](https://beam.apache.org/get-started/wordcount-example/) for most
[data](https://ci.apache.org/projects/flink/flink-docs-release-1.0/apis/batch/examples.html#word-count)
[processing](https://spark.apache.org/examples.html)
[frameworks](https://github.com/apache/storm/blob/manager/examples/storm-starter/src/jvm/org/apache/storm/starter/WordCountTopology.java).

The `data` directory has a collection of texts downloaded from [Project
Gutenberg](https://www.gutenberg.org/ebooks/search/%3Fsort_order%3Ddownloads)
(`data/pg-*`).

An implementation of word count has been started for you in `wordcount/main.go`.
Your job is to write the map (`mapFn`) and reduce (`reduceFn`) functions.

In doing so, you might find the following resources helpful:
* [Go blog on strings](http://blog.golang.org/strings)
* [`strings.FieldsFunc`](http://golang.org/pkg/strings/#FieldsFunc)
* [`strconv` package](http://golang.org/pkg/strconv/)
* [`unicode.IsLetter`](http://golang.org/pkg/unicode/#IsLetter)

When you are done, you can run a sequential MapReduce:
```
go run mapreduce/wordcount data/pg-metamorphosis.txt
```

You can see the 10 most frequent words by doing the following:
```
$ sort -n -k2 mrtmp.wordcount-sequential | tail -10
that: 343
had: 350
in: 395
was: 407
he: 508
his: 524
of: 541
and: 680
to: 827
the: 1265
```

You can inspect the entire output with `less mrtmp.wordcount-sequential`
(hitting `q` on the keyboard quits).

You can test your complete solution with the provided script, which diffs a
segment of your output against a reference:
```
./test-wc.sh
```

Running the cleanup script will remove all output of tests:
```
./cleanup.sh
```

Make sure that you commit and push your changes after finishing this part of the
lab.

## Part 3: MapReduce Distributed Manager

The sequential MapReduce runs all map/reduce tasks sequentially, one after
another. Since the main point of MR is to make distributed data processing
easier, we should implement a distributed manager.

At the completion of this part of the lab, the Parallel unit tests should now
pass.
```
go test mapreduce/mapreduce -run Parallel -test.v
```

Most of the framework for the distributed version has been provided for you. The
manager (`mapreduce/manager.go`) RPC interface for worker registration and all
the logic required to run the phases. The worker (`mapreduce/worker.go`) can
register with the manager and has an RPC interface to do tasks (by calling your
code from Part 1).

For this part of the lab, you will implement `executePhase` in
`mapreduce/phase_executor.go`.

The entry point to start a phase is `executePhase`. This function is responsible
for scheduling all work. I have provided you with an implementation that
delegates work to a function called called `match` that schedules work by
performing a matching of tasks to workers.

`match` is given a `phaseJob` struct, which contains `TaskQueue` (`chan int`)
and `WorkerQueue` (`chan string`) channels. The task queue channel is initially
full of all work to be done, and the worker queue is populated as workers
register.

As workers register, the `match` function should find idle workers and give them
a task (using the `call_{map,reduce}` functions). Note that when a worker
finishes, your code will have to handle making sure that worker is eligible to
be matched again. For this part you don't have to worry about fault tolerance
yet.

Your function must also make sure that it is not running all tasks sequentially.
Workers __must__ be able to process tasks concurrently. There is a unit test
(specifically, `TestParallelCheck`) that validates that this is the case.

Run the tests to validate your Part 3 solution.
```
go test mapreduce/mapreduce -run Parallel -test.v
```

When you are done, you should also validate that your code does not have any
races by running Go's built-in [race
detector](https://golang.org/doc/articles/race_detector.html)
```
go test mapreduce/mapreduce -run Parallel -test.v -race
```

At this point, you can now run a distributed MapReduce. First start the manager.
```
go run mapreduce/wordcount -logtostderr -distributed data/pg-*.txt
```

Then run a worker (in a different terminal)!
```
go run mapreduce/wordcount -logtostderr -worker 
```

You can test your manager with multiple workers by having them have different RPC
addresses (you'll need yet another terminal):
```
go run mapreduce/wordcount -logtostderr -worker -workerAddr localhost:7777
go run mapreduce/wordcount -logtostderr -worker -workerAddr localhost:7778
```

There is also a test script that will start the MapReduce manager and three
workers: `test-wc-distributed.sh`:
```
./test-wc-distributed.sh
```

Again, run the cleanup script to clean up your temporary files.
```
./cleanup.sh
```

Don't forget to commit and push your code!

## Part 4: Fault Tolerance

If your `match` function supports failing workers, you're done. Otherwise, your
job is to now ensure that `match` is fault tolerant.

When a worker fails, eventually the `DoWork` RPC will time out, and
`call_{map,reduce}` will return false. When this happens, you must make sure
that the task is re-assigned from the failed worker to another worker.

Note that in the real world the RPC may fail for any number of reasons and it is
possible to have two workers working on the same task. In the MapReduce paper,
this is handled by having the manager have only one view of the output files
produced, and giving this view to the reducer (in the case of mapper failure),
or by using atomic rename (reducer failure). We could do something similar
here, but will skip handling this type of failure. You also don't have to worry
about handling failure of the manager.

Now, at this point, the Failure tests (and indeed all tests) should pass.
```
go test mapreduce/mapreduce -run Failure -test.v
go test mapreduce/mapreduce
```

## Part 5: Inverted Index

In this part of the lab, you use MapReduce to build an [_inverted
index_](https://en.wikipedia.org/wiki/Inverted_index). An inverted index is an
inversion of a key-value map: it is a map that maps all values in the original
map to the keys where those values appear.

For example, for word count, this would be a mapping from frequencies to words
that appeared with that frequency:
```
89: bound, horror, question, third
```

A frequent use of an inverted index is in search where an inverted index stores
a map from terms to a list of documents that contain that term.

This part of the lab is optional, and will be extra credit.

There is a skeleton implementation of a MapReduce for generating an inverted
index in `invertedindex/main.go`. You can see sample output in
`data/ii-testout.txt`. Complete the Map and Reduce functions for the
inverted index generation. You can generate an inverted index using:
```
go run mapreduce/invertedindex data/pg-metamorphosis.txt
```

Your generated output should map words to files where they occurred:
```
$ head -n10 mrtmp.ii-sequential
A: 3 data/pg-metamorphosis.txt,data/pg-grimm.txt,data/pg-sherlock_holmes.txt
ACTUAL: 3 data/pg-metamorphosis.txt,data/pg-grimm.txt,data/pg-sherlock_holmes.txt
ADLER: 1 data/pg-sherlock_holmes.txt
ADVENTURE: 1 data/pg-sherlock_holmes.txt
ADVENTURES: 2 data/pg-grimm.txt,data/pg-sherlock_holmes.txt
AGREE: 3 data/pg-metamorphosis.txt,data/pg-grimm.txt,data/pg-sherlock_holmes.txt
AGREEMENT: 3 data/pg-metamorphosis.txt,data/pg-grimm.txt,data/pg-sherlock_holmes.txt
AK: 3 data/pg-metamorphosis.txt,data/pg-grimm.txt,data/pg-sherlock_holmes.txt
AND: 3 data/pg-metamorphosis.txt,data/pg-grimm.txt,data/pg-sherlock_holmes.txt
ANY: 3 data/pg-metamorphosis.txt,data/pg-grimm.txt,data/pg-sherlock_holmes.txt
```

Again, you can view your output with `less mrtmp.ii-sequential` (hit `q` to
quit).

Run the provided script to test your implementation against reference data.
```
./test-ii.sh
```

There is also a test script that will start the MapReduce manager and three
workers: `test-ii-distributed.sh`:
```
./test-ii-distributed.sh
```

Make sure when you are done you commit and push your changes.

# Congratulations! You are done!
