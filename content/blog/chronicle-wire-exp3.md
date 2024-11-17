+++
title = 'Chronicle Wire with gRPC'
date = '2024-11-17T21:14:08+05:30'
draft = false
+++


# Integrating Chronincle Wire within GRPC based backend 

## Intro
Recently, I got introduced to [chronicle wire](http://chronicle.software/chronicle-wire-object-marshalling/0) as a high performance serialization library. While [gRPC](http://grpc.io) based concurrency/latency benefits had always been on top of my mind, I wanted to put theories to test. So, I married the two tasks together. Voila, the ask is to test for a custom grpc backend not utilizing protocol buffers serialization for transport, but chronicle wire.

## Setup
As always, my old MBP 2016, no external cloud server setup. 

This time, we need a maven project setup to manage dependencies with ease. I fancied myself with slf4j backed with Java Util Logging library, though it is not requirement for the experiment.
The project is run via an executable uber jar ( maven-shade-plugin at play). 

The source code for this experiment is synced/hosted at [Github](https://github.com/sumit-mundra/cwire-grpc-example). 

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


```
Started
Clients, RPCs, RPCs/s, sum(us), avg(us), simulated_delay(us), delta(us)
<<omitted>>
cwire-grpc-example-1.0-SNAPSHOT.jar
   1,  20388,  2038.80,  9764395.00,   478.93,    0,   478.93
  10,  42133,  4213.30,  9723696.00,   230.79,    0,   230.79
 100,  41762,  4176.20,  9758782.00,   233.68,    0,   233.68
   1,   5271,   527.10,  9936468.00,  1885.12,    1,  1884.12
  10,   5433,   543.30,  9934307.00,  1828.51,    1,  1827.51
 100,   5369,   536.90,  9944015.00,  1852.12,    1,  1851.12
   1,   5613,   561.30,  9941261.00,  1771.11,   10,  1761.11
  10,   5475,   547.50,  9941111.00,  1815.73,   10,  1805.73
 100,   5554,   555.40,  9936992.00,  1789.16,   10,  1779.16
   1,   5564,   556.40,  9939306.00,  1786.36,  100,  1686.36
  10,   5582,   558.20,  9939556.00,  1780.64,  100,  1680.64
 100,   5462,   546.20,  9937968.00,  1819.47,  100,  1719.47
   1,   5432,   543.20,  9940905.00,  1830.06, 1000,   830.06
  10,   5428,   542.80,  9940311.00,  1831.30, 1000,   831.30
 100,   5438,   543.80,  9938362.00,  1827.58, 1000,   827.58
 ```

The output text omits output line about chronicle wire library version )
`Chronicle core loaded from file: ... cwire-grpc-example-1.0-SNAPSHOT.jar`

We could observe that we are reaping dual benefits of gRPC parallel execution over netty. I am not sure about the serialization benefits at this scale here though and would need another experiment with gson as serialization for a comparison. Please wait till the next post comes out ...