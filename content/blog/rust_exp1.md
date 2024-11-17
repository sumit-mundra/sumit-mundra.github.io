+++
title = 'Fibonacci Race between Rust And JVM Execution'
date = '2024-10-12T20:18:41+05:30'
draft = false
+++

# Fibonacci in Rust and Java
I was curious to compare the runtimes between java and rust. This benchmark exercise was done on my MBP 2016 model with not much care around running programs. Given that the two programs executed back to back, I doubt any OS level resources congestion would have made any difference to the results.

## Setup
For my study purpose, I was trying to write fibonacci series program with rust. So there are two versions to compare fibonacci implementation: 

- Recursive Implementation
- Iterative Implementation

Both the approach were implemented in `java` and `rust` to get a comparison.

## Results
1. Rust >>> JVM : I got surprised that JVM took a lot of time for iterative approach with 0th item, which is a straight lookup.
However, rust was super fluent all along. A native binary execution beat the execution on a virtual machine (jvm) -- (no warmup done for either code).

2. Warmup: JVM does some optimisation on-the-fly for methods repeatedly executed as well. And, an initial delay of 5s post boot might further reduce 0th iteration values. Even then, the rust execution results beat jvm results by a far margin.

### Appendix:
#### Rust Program: 
```rust
use std::time::Instant;

fn main() {
    for i in 0..=40 {
        let a = Instant::now();
        n_fib(i);
        let b = a.elapsed().as_nanos();
        n_fib_iter(i);
        let c = a.elapsed().as_nanos() - b;
        println!("({}) : ({:?},{:?})", i, b, c);
    }
}

fn n_fib(n: u32) -> u32 {
    match n {
        0 => 0,
        1 => 1,
        _ => n_fib(n - 2) + n_fib(n - 1)
    }
}

fn n_fib_iter(n: u32) -> u32 {
    match n {
        0 => n,
        1 => n,
        _ => {
            let mut a = 0;
            let mut b = 1;
            for _ in 1..=n {
                let temp = a + b;
                a = b;
                b = temp;
            }
            b
        }
    }
}
```
**Output**:

```
(0) : (114,111)
(1) : (71,80)
(2) : (102,106)
(3) : (104,114)
(4) : (117,118)
(5) : (157,117)
(6) : (205,133)
(7) : (257,156)
(8) : (269,86)
(9) : (350,91)
(10) : (500,105)
(11) : (666,95)
(12) : (882,104)
(13) : (1185,97)
(14) : (2179,127)
(15) : (3843,94)
(16) : (5998,112)
(17) : (8259,104)
(18) : (13757,122)
(19) : (20500,132)
(20) : (37229,142)
(21) : (60543,173)
(22) : (92234,150)
(23) : (138869,150)
(24) : (186743,161)
(25) : (316862,160)
(26) : (459290,148)
(27) : (740209,116)
(28) : (1192564,159)
(29) : (1924402,265)
(30) : (3070848,205)
(31) : (5398369,229)
(32) : (8003983,190)
(33) : (12163061,290)
(34) : (21837659,396)
(35) : (34445105,245)
(36) : (52977608,192)
(37) : (79761066,164)
(38) : (127993165,326)
(39) : (220469246,192)
(40) : (356705776,326)

```

#### Java Program: 
```java
import java.time.Duration;
import java.time.ZonedDateTime;

class Scratch {
    public static void main(String[] args) {
        for (int i = 0; i <= 40; i++) {
            ZonedDateTime time = ZonedDateTime.now();
            fib(i);
            long delta1 = Duration.between(time, ZonedDateTime.now()).toNanos();
            fibIterative(i);
            long delta2 = Duration.between(time, ZonedDateTime.now()).toNanos();
            System.out.print(i + " :: " + delta1 + ", " + (delta2 - delta1) + "\n");
        }
    }

    public static int fibIterative(int n) {
        if (n == 0 || n == 1)
            return n;
        int a = 0;
        int b = 1;
        for (int i = 1; i < n; i++) {
            int temp = b;
            b = a + b;
            a = temp;
        }
        return b;
    }

    public static int fib(int n) {
        if (n == 0)
            return 0;
        if (n == 1)
            return 1;
        return fib(n - 2) + fib(n - 1);
    }


}
```
**Output**:

```
0 :: 228000, 1269000
1 :: 21000, 27000
2 :: 11000, 15000
3 :: 9000, 15000
4 :: 10000, 15000
5 :: 9000, 15000
6 :: 11000, 14000
7 :: 12000, 19000
8 :: 19000, 22000
9 :: 28000, 22000
10 :: 23000, 20000
11 :: 53000, 22000
12 :: 44000, 21000
13 :: 59000, 30000
14 :: 66000, 23000
15 :: 33000, 65000
16 :: 57000, 24000
17 :: 57000, 24000
18 :: 65000, 17000
19 :: 72000, 14000
20 :: 136000, 18000
21 :: 346000, 43000
22 :: 406000, 42000
23 :: 294000, 29000
24 :: 323000, 34000
25 :: 496000, 23000
26 :: 817000, 31000
27 :: 1287000, 29000
28 :: 2072000, 32000
29 :: 3432000, 45000
30 :: 5461000, 47000
31 :: 9854000, 57000
32 :: 14527000, 44000
33 :: 22957000, 45000
34 :: 38971000, 51000
35 :: 58531000, 43000
36 :: 95962000, 44000
37 :: 167077000, 56000
38 :: 238931000, 43000
39 :: 375188000, 56000
40 :: 559389000, 54000
```