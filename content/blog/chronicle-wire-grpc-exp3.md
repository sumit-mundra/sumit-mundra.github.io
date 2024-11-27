+++
title = 'Integrating Chronincle Wire with GRPC backend'
date = '2024-11-17T21:14:08+05:30'
draft = false
+++


# Intro

Recently, I got introduced to [chronicle wire](http://chronicle.software/chronicle-wire-object-marshalling/0) as a high performance serialization library. While [gRPC](http://grpc.io) based concurrency/latency benefits had always been on top of my mind, I wanted to put theories to test. So, I married the two tasks together. Voila, the ask is to test for a custom grpc backend not utilizing protocol buffers serialization for transport, but chronicle wire.

## Setup
As always, my old MBP 2016, no external cloud server setup. 

This time, we need a maven project setup to manage dependencies with ease. I fancied myself with slf4j backed with Java Util Logging library, though it is not requirement for the experiment.
The project is run via an executable uber jar ( maven-shade-plugin at play). 

*The source code for this experiment is synced/hosted at [Github](https://github.com/sumit-mundra/cwire-grpc-example).*

To run at your local, clone the repo and cd into the repo directory
``` sh
mvn clean package
java -jar target/cwire-grpc-example-1.0-SNAPSHOT.jar
```


## Method
There are several key concerns within this implementation: 
* To integrate a custom serializer without .proto defined into gRPC framework, you need to declare a service definition as hinted in this [blog](https://grpc.io/blog/grpc-with-json/). `ChronicleWireMarshaller` and `ChronicleWireServiceDefinition` take care of this.
* The client makes unary blocking RPC (ie one RPC per client at a time) to allow for better control on client concurrency externally. The client count is maintained in `Runner`.
* To allow for accuracy of latency measurements, the work has been delegated to an executor service. `HelloServiceClient`
* `Runner` initiates Server with a simulated `delay` and also instantiates `HelloServiceClient` with a managed channel for every iteration.
* For each (client concurrency, delay) pair, an unbound execution loop is run while metrics is measured in parallel. `Runner#runClient`

## Results
The data below shows 
- Clients: Max parallel client connections
- RPCs: Procedure Calls made in total duration ( 10s here).
- RPC/s: Total RPC / Total Duration
- Sum (us): Sum of total latency in microseconds
- Avg(us): Average latency ie Sum(us)/Total RPCs
- Simulated_delay(us): Server side simulated delay added by thread sleep in each procedure call. (in microseconds)
- Delta(us): Average delta per call (in microseconds) added because of other code executions like serialization, gc etc. (Sum - (simulated delay * RPCs))/RPCs

In a 10s run, on a 8 cpu threads with under 50% cpu usage, we were able to reach `~600 RPC/s`. Also, the average platform delay hovered around `600-1600 us`. The cpu usage stayed under 25% on my mac for all cases with simulated delay, while in initial no delay, it spiked up to 140% ( base 800%).

``` text
Started
Clients, RPCs, RPCs/s, sum(us), avg(us), simulated_delay(us), delta(us)
<<omitted>>
cwire-grpc-example-1.0-SNAPSHOT.jar
   1,  27627,  2762.70,  9766997.00,   353.53,    0,   353.53
  10,  60806,  6080.60,  9728751.00,   160.00,    0,   160.00
 100,  61799,  6179.90,  9762235.00,   157.97,    0,   157.97
   1,   6066,   606.60,  9954892.00,  1641.10,    1,  1640.10
  10,   6150,   615.00,  9954252.00,  1618.58,    1,  1617.58
 100,   6164,   616.40,  9951140.00,  1614.40,    1,  1613.40
   1,   6138,   613.80,  9954634.00,  1621.80,   10,  1611.80
  10,   6044,   604.40,  9949912.00,  1646.25,   10,  1636.25
 100,   5943,   594.30,  9945836.00,  1673.54,   10,  1663.54
   1,   6090,   609.00,  9948296.00,  1633.55,  100,  1533.55
  10,   6116,   611.60,  9947999.00,  1626.55,  100,  1526.55
 100,   6083,   608.30,  9928227.00,  1632.13,  100,  1532.13
   1,   6013,   601.30,  9954174.00,  1655.44, 1000,   655.44
  10,   6015,   601.50,  9960095.00,  1655.88, 1000,   655.88
 100,   6105,   610.50,  9950295.00,  1629.86, 1000,   629.86
 ```

The output text omits output line about chronicle wire inclusion path
`Chronicle core loaded from file: .../cwire-grpc-example-1.0-SNAPSHOT.jar`

We could observe that we are reaping dual benefits of gRPC parallel execution over netty. I am not sure about the serialization benefits at this scale here though and would need another experiment with gson as serialization for a comparison. Please wait till the [next post](../gson-proto-chroniclewire-exp4/) comes out ...