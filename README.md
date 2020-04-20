<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# Flink Training Exercises

Exercises that go along with the training content in the documentation.

## Table of Contents

[**Setup your Development Environment**](#setup-your-development-environment)

1. [Software requirements](#software-requirements)
1. [Clone and build the flink-training project](#clone-and-build-the-flink-training-project)
1. [Import the flink-training project into your IDE](#import-the-flink-training-project-into-your-ide)
1. [Download the data sets](#download-the-data-sets)

[**Using the Taxi Data Streams**](#using-the-taxi-data-streams)

1. [Schema of Taxi Ride Events](#schema-of-taxi-ride-events)
1. [Generating Taxi Ride Data Streams in a Flink program](#generating-taxi-ride-data-streams-in-a-flink-program)

[**How to do the Labs**](#how-to-do-the-labs)

1. [Learn about the data](#learn-about-the-data)
1. [Modify `ExerciseBase`](#modify-exercisebase)
1. [Run and debug Flink programs in your IDE](#run-and-debug-flink-programs-in-your-ide)
1. [Exercises, Tests, and Solutions](#exercises-tests-and-solutions)

[**Labs**](LABS-OVERVIEW.md)

## Setup your Development Environment

The following instructions guide you through the process of setting up a development environment for the purpose of developing, debugging, and executing solutions to the Flink developer training exercises and examples.

### Software requirements

Flink supports Linux, OS X, and Windows as development environments for Flink programs and local execution. The following software is required for a Flink development setup and should be installed on your system:

- a JDK for Java 8 or Java 11 (a JRE is not sufficient; other versions of Java are not supported)
- Git
- an IDE for Java (and/or Scala) development with Gradle support.
  We recommend IntelliJ, but Eclipse or Visual Studio Code can also be used so long as you stick to Java. For Scala you will need to use IntelliJ (and its Scala plugin).

> **:information_source: Note for Windows users:** Many of the examples of shell commands provided in the training instructions are for UNIX systems. To make things easier, you may find it worthwhile to setup cygwin or WSL. For developing Flink jobs, Windows works reasonably well: you can run a Flink cluster on a single machine, submit jobs, run the webUI, and execute jobs in the IDE.

### Clone and build the flink-training project

This `flink-training` project contains exercises, tests, and reference solutions for the programming exercises. Clone the `flink-training` project from Github and build it.

```bash
git clone https://github.com/apache/flink-training.git
cd flink-training
./gradlew test shadowJar
```

If you haven’t done this before, at this point you’ll end up downloading all of the dependencies for this Flink training project. This usually takes a few minutes, depending on the speed of your internet connection.

If all of the tests pass and the build is successful, you are off to a good start.

<details>
<summary><strong>Users in China: click here for instructions about using a local maven mirror.</strong></summary>

If you are in China, we recommend configuring the maven repository to use a mirror. You can do this by uncommenting the appropriate line in our [`build.gradle`](build.gradle) like this:

```groovy
    repositories {
        // for access from China, you may need to uncomment this line
        maven { url 'http://maven.aliyun.com/nexus/content/groups/public/' }
        mavenCentral()
    }
```
</details>


### Import the flink-training project into your IDE

The project needs to be imported as a gradle project into your IDE.

Once that’s done you should be able to open [`RideCleansingTest`](ride-cleansing/src/test/java/org/apache/flink/training/exercises/ridecleansing/RideCleansingTest.java) and successfully run this test.

> **:information_source: Note for Scala users:** You will need to use IntelliJ with the JetBrains Scala plugin, and you will need to add a Scala 2.12 SDK to the Global Libraries section of the Project Structure. IntelliJ will ask you for the latter when you open a Scala file.

### Download the data sets

You will also need to download the taxi data files used in this training by running the following commands

```bash
wget http://training.ververica.com/trainingData/nycTaxiRides.gz
wget http://training.ververica.com/trainingData/nycTaxiFares.gz
```

It doesn’t matter if you use `wget` or something else (like `curl`, or Chrome) to download these files, but however you get the data, **do not decompress or rename the `.gz` files**. Some browsers will do the wrong thing by default.

> **:information_source: Note:** There are hardwired paths to these data files in the exercises, in `org.apache.flink.training.exercises.common.utils.ExerciseBase`. The tests don’t use these data files, but before running the exercises, you will need to fix these paths to point to the files you have downloaded.

## Using the Taxi Data Streams

The [New York City Taxi & Limousine Commission](http://www.nyc.gov/html/tlc/html/home/home.shtml) provides a public [data set](https://uofi.app.box.com/NYCtaxidata) about taxi rides in New York City from 2009 to 2015. We use a modified subset of this data to generate streams of events about taxi rides. You should have downloaded these in the [steps above](#download-the-data-sets).

### Schema of Taxi Ride Events

Our taxi data set contains information about individual taxi rides in New York City. Each ride is represented by two events: a trip start, and a trip end event. Each event consists of eleven fields:

```
rideId         : Long      // a unique id for each ride
taxiId         : Long      // a unique id for each taxi
driverId       : Long      // a unique id for each driver
isStart        : Boolean   // TRUE for ride start events, FALSE for ride end events
startTime      : DateTime  // the start time of a ride
endTime        : DateTime  // the end time of a ride,
                           //   "1970-01-01 00:00:00" for start events
startLon       : Float     // the longitude of the ride start location
startLat       : Float     // the latitude of the ride start location
endLon         : Float     // the longitude of the ride end location
endLat         : Float     // the latitude of the ride end location
passengerCnt   : Short     // number of passengers on the ride
```

> **:information_source: Note:** The data set contains records with invalid or missing coordinate information (longitude and latitude are 0.0).

There is also a related data set containing taxi ride fare data, with these fields:

```
rideId         : Long      // a unique id for each ride
taxiId         : Long      // a unique id for each taxi
driverId       : Long      // a unique id for each driver
startTime      : DateTime  // the start time of a ride
paymentType    : String    // CSH or CRD
tip            : Float     // tip for this ride
tolls          : Float     // tolls for this ride
totalFare      : Float     // total fare collected
```

### Generating Taxi Ride Data Streams in a Flink program

> **:information_source: Note: The exercises already provide code for working with these taxi ride data streams.**

We provide a Flink source function (`TaxiRideSource`) that reads a `.gz` file with taxi ride records and emits a stream of `TaxiRide` events. The source operates in [event-time](https://ci.apache.org/projects/flink/flink-docs-stable/dev/event_time.html). There’s an analogous source function (`TaxiFareSource`) for `TaxiFare` events.

In order to generate these streams as realistically as possible, events are emitted proportional to their timestamp. Two events that occurred ten minutes after each other in reality are also served ten minutes after each other. A speed-up factor can be specified to “fast-forward” the stream, i.e., given a speed-up factor of `60`, events that happened within one minute are served in one second. Moreover, one can specify a maximum serving delay which causes each event to be randomly delayed within the specified bound. This yields an out-of-order stream as is common in many real-world applications.

For these exercises, a speed-up factor of `600` or more (i.e., 10 minutes of event time for every second of processing), and a maximum delay of `60` (seconds) will work well.

All exercises should be implemented using event-time characteristics. Event-time decouples the program semantics from serving speed and guarantees consistent results even in case of historic data or data which is delivered out-of-order.

#### How to use these sources

**Java**

```java
// get an ExecutionEnvironment
StreamExecutionEnvironment env =
  StreamExecutionEnvironment.getExecutionEnvironment();
// configure event-time processing
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

// get the taxi ride data stream
DataStream<TaxiRide> rides = env.addSource(
  new TaxiRideSource("/path/to/nycTaxiRides.gz", maxDelay, servingSpeed));
```

**Scala**

```scala
// get an ExecutionEnvironment
val env = StreamExecutionEnvironment.getExecutionEnvironment
// configure event-time processing
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

// get the taxi ride data stream
val rides = env.addSource(
  new TaxiRideSource("/path/to/nycTaxiRides.gz", maxDelay, servingSpeed))
```

There is also a `TaxiFareSource` that works in an analogous fashion, using the `nycTaxiFares.gz` file. This source creates a stream of `TaxiFare` events.

**Java**

```java
// get the taxi fare data stream
DataStream<TaxiFare> fares = env.addSource(
  new TaxiFareSource("/path/to/nycTaxiFares.gz", maxDelay, servingSpeed));
```

**Scala**

```scala
// get the taxi fare data stream
val fares = env.addSource(
  new TaxiFareSource("/path/to/nycTaxiFares.gz", maxDelay, servingSpeed))
```

## How to do the Labs

In the hands-on sessions you will implement Flink programs using various Flink APIs.

The following steps guide you through the process of using the provided data streams, implementing your first Flink streaming program, and executing your program in your IDE.

We assume you have setup your development environment according to our [setup guide above](#setup-your-development-environment).

### Learn about the data

The initial set of exercises are all based on data streams of events about taxi rides and taxi fares. These streams are produced by source functions which reads data from input files. Please read the [instructions above](#using-the-taxi-data-streams) to learn how to use them.

### Modify `ExerciseBase`

After downloading the datasets, open the `org.apache.flink.training.exercises.common.utils.ExerciseBase` class in your IDE, and edit these two lines to point to the two taxi ride data files you have downloaded:

```java
public final static String pathToRideData =   
    "/Users/david/stuff/flink-training/trainingData/nycTaxiRides.gz";
public final static String pathToFareData =
    "/Users/david/stuff/flink-training/trainingData/nycTaxiFares.gz";
```

### Run and debug Flink programs in your IDE

Flink programs can be executed and debugged from within an IDE. This significantly eases the development process and provides an experience similar to working on any other Java (or Scala) application.

Starting a Flink program in your IDE is as easy as running its `main()` method. Under the hood, the execution environment will start a local Flink instance within the same process. Hence it is also possible to put breakpoints in your code and debug it.

Assuming you have an IDE with this `flink-training` project imported, you can run (or debug) a simple streaming job as follows:

- Open the `org.apache.flink.training.examples.ridecount.RideCountExample` class in your IDE
- Run (or debug) the `main()` method of the `RideCountExample` class using your IDE.

### Exercises, Tests, and Solutions

Many of the exercises include an `...Exercise` class with most of the necessary boilerplate code for getting started, as well as a JUnit Test class (`...Test`) with a few tests for your implementation, and a `...Solution` class with a complete solution.

> **:information_source: Note:** As long as your `...Exercise` class is throwing a `MissingSolutionException`, the provided JUnit test classes will ignore that failure and verify the correctness of the solution implementation instead.

-----

Now you are ready to begin with the first exercise in our [**Labs**](LABS-OVERVIEW.md).