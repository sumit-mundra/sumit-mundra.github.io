+++
title = 'Rust vs Java Concurrent Hashmap'
date = '2024-10-25T17:47:16+05:30'
draft = false
+++

# Race 2: Concurrent Updates to a Mutexed Hashmap in Rust and Java
Very often, we find ourselves using an in-memory cache implementation which spans across thread boundaries in an application. This experiment tries to evaluate a small implementation using hashmaps with "mutually exclusive" operations.

## Setup
* The host machine is running on Macbook Pro 2016 model with Intel i7 processor, 16GB RAM ( 8 threads ). There were no other demanding programs other than IDE/terminals needed. The amazon-correto-jdk21 and rust 1.81.0 were used to execute these test runners. 
* For exploration, an integer-keyed hashmap with custom data like a customer struct/object is populated with customer id as unique identifier. The data is stored in the heap memory via default hashmap implementation. For concurrency, both read and write accesses to the map were async and exectuted via a spawned OS thread under same process. In both the cases, the random number generation was part of the thread creation loop.

## Results

1. With java, the results were increasingly better with higher concurrency, whereas with rust, the results went poorer. However, they were better or comparable at lower concurrency.
2. Hasher plays an important part in hashmap performance. For rust, default hasher is suitable to prevent DoS attacks, while it pays cost for that even for simple insertions.

## Observations
1. There is a limit of max threads that can spawn under macOS. Somehow, rust peak concurrency was not observed beyond 5 even though the overall runtime was way beyond many seconds.
2. OS threads spawned by JVM platform quickly.
3. There are faster hashmap implementations within rust community and even runtimes for virtual thread (like tokio and rayon ). But, including them was beyond the scope for this experiment.


## Source Code
### Rust
[src](https://raw.githubusercontent.com/sumit-mundra/exp_hashmap/refs/heads/fxhash/src/main.rs)

``` rust
use std::thread;
use std::thread::JoinHandle;
use std::time::{Duration, Instant};

use rand::{thread_rng, Rng};
use server::{Customer, CustomerService};

fn main() {
    println!("Start!! {}", thread::available_parallelism().unwrap());
    let start = Instant::now();
    let server = CustomerService::new();
    // let mut write_handles = Vec::new();
    const MAX_ID_VALUE: isize = 10000;
    const OPERATIONS: usize = 10000;
    const DURATION_ZERO: Duration = Duration::from_secs(0);
    let mut rd_ar: [JoinHandle<Duration>; OPERATIONS] =
        core::array::from_fn(|_| thread::spawn(|| DURATION_ZERO));
    let mut wr_ar: [JoinHandle<Duration>; OPERATIONS] =
        core::array::from_fn(|_| thread::spawn(|| DURATION_ZERO));

    for i in 0..OPERATIONS {
        let j = thread_rng().gen_range(0..MAX_ID_VALUE);
        // write_handles.push(server.upsert_async(Customer::new(k, &format!("Kamal {}", &k), "Hasan")));
        let t = Instant::now();
        let name = &format!("Foo {}", &j);
        wr_ar[i] = server.upsert_async(Customer::new(j, name, "Bar"), t);
    }

    for ind in 0..OPERATIONS {
        let k = thread_rng().gen_range(0..MAX_ID_VALUE);
        let t2 = Instant::now();
        rd_ar[ind] = server.print_async(k, t2);
    }
    let total_reads = f64::from(rd_ar.len() as u32);
    let total_writes = f64::from(wr_ar.len() as u32);
    let mut total_write_duration = Duration::from_micros(0);
    for handle in wr_ar {
        total_write_duration += handle.join().unwrap();
    }
    println!("After all write handles threads have finished");
    println!("Finished writes in {:?}", Instant::now() - start);
    let mut total_read_duration = Duration::from_micros(0);
    for handle in rd_ar {
        total_read_duration += handle.join().unwrap();
    }
    println!("After all read handles threads have finished");
    println!("Finished reads in {:?}", Instant::now() - start);
    println!(
        "For {} cycles, Total read duration {}ms Total write duration {}ms",
        OPERATIONS,
        total_read_duration.as_millis(),
        total_write_duration.as_millis()
    );
    println!(
        "Average read {} ops/s, write {} ops/s",
        total_reads / (total_read_duration.as_secs_f64()),
        total_writes / (total_write_duration.as_secs_f64())
    );
    println!("Finished in {:?}", Instant::now() - start);
}

/// This module is intended to be backend managing map.
/// allowing concurrent access to hashmap and update it
/// map contains <customer id, customer data object>
mod server {
    use std::collections::HashMap;

    use std::sync::{Arc, Mutex};
    use std::thread;
    use std::thread::JoinHandle;
    use std::time::{Duration, Instant};

    pub struct CustomerService {
        store: Arc<Mutex<HashMap<isize, Customer>>>,
    }

    impl CustomerService {
        pub fn new() -> CustomerService {
            CustomerService {
                store: Arc::new(Mutex::new(HashMap::default())),
            }
        }

        pub fn upsert_async(&self, customer: Customer, instant: Instant) -> JoinHandle<Duration> {
            // println!("Inside upsert");
            let arc = Arc::clone(&self.store);
            thread::spawn(move || {
                loop {
                    match arc.try_lock() {
                        Ok(mut guard) => {
                            // println!("thread");
                            let _c = guard.insert(customer.id, customer);
                            return instant.elapsed();
                        }
                        Err(_) => {
                            thread::sleep(Duration::from_millis(100));
                            continue;
                        }
                    }
                }
                // println!(".");
            })
        }

        pub fn print_async(&self, id: isize, instant: Instant) -> JoinHandle<Duration> {
            let arc = Arc::clone(&self.store);
            thread::spawn(move || {
                loop {
                    match arc.try_lock() {
                        Ok(guard) => {
                            let _c = guard.get(&id);
                            return instant.elapsed();
                        }
                        Err(_) => {
                            thread::sleep(Duration::from_millis(1));
                            continue;
                        }
                    }
                }
                //match opt {
                //     None => {
                //         //println!("Not yet filled {}", id)
                //     },
                //     Some(_customer) => {
                //         // _customer.print();
                //     }
                // };
            })
        }
    }

    /**
    This struct holds basic customer data of first name and last name with a numeric identifier.
    This struct does not guarantee uniqueness checks on creation
    */
    pub struct Customer {
        pub id: isize,
        pub first_name: String,
        pub last_name: String,
    }

    impl Customer {
        pub fn new(id: isize, first_name: &str, last_name: &str) -> Customer {
            Customer {
                id,
                first_name: first_name.to_string(),
                last_name: last_name.to_string(),
            }
        }

        /// To print the customer struct on console
        pub fn print(&self) {
            println!(
                "id: {}, FN: {}, LN: {}",
                self.id, self.first_name, self.last_name
            )
        }
    }
}
```
### Java
[src](https://github.com/sumit-mundra/exp_hashmap/raw/refs/heads/main/src/HashMapRunner.java)
``` java
import java.util.*;
import java.util.concurrent.*;


class HashMapRunner {
    static final ExecutorService executor = Executors.newCachedThreadPool();
    private static final int MAX_ID_VALUE = 10000;
    private static final int MAX_LOOP = 10000;

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Thread.sleep(5000);
        long startSystem = System.nanoTime();
        final CustomerService service = new CustomerService(MAX_ID_VALUE);
        List<Future<Long>> readFutures = new ArrayList<>(MAX_LOOP);
        List<Future<Long>> writeFutures = new ArrayList<>(MAX_LOOP);
        Random random = new Random(5);
        for (int i = 0; i < MAX_LOOP; i++) {
            Integer k = random.nextInt(MAX_ID_VALUE);
            writeFutures.add(service.upsert(k, new Customer(k, "Foo" + k, "Bar")));
        }
        for (int i = 0; i < MAX_LOOP; i++) {
            Integer l = random.nextInt(MAX_ID_VALUE);
            readFutures.add(service.readCustomer(l));
        }
        Long writeTotal = 0L;
        Long readTotal = 0L;
        for (Future<Long> writeFuture : writeFutures) {
            Long data = writeFuture.get();
            writeTotal += data;
        }
        System.out.println("After all write futures have finished");
        for (Future<Long> readFuture : readFutures) {
            Long data = readFuture.get();
            readTotal += data;
        }
        System.out.println("After all read futures have finished");
        System.out.println("readTotal = " + readTotal + ", writeTotal = " + writeTotal);
        System.out.printf("For %d cycles, Total read duration %.2f s Total write duration %.2f s \n", MAX_LOOP, readTotal / (1000 * 1000 * 1000f), writeTotal / (1000 * 1000 * 1000f));
        System.out.printf("Average read %.2f ops/s, write %.2f ops/s\n", (Integer.valueOf(MAX_LOOP).doubleValue() * 1000 * 1000 * 1000) / readTotal, (Integer.valueOf(MAX_LOOP).doubleValue() * 1000 * 1000 * 1000) / writeTotal);
        executor.shutdown();
        System.out.println("Duration::" + (System.nanoTime() - startSystem));
    }

    static class Customer {
        Integer id;
        String firstName;
        String lastName;

        public Customer(Integer id, String firstName, String lastName) {
            this.id = id;
            this.firstName = firstName;
            this.lastName = lastName;
        }

        @Override
        public String toString() {
            return "id: " + this.id + ", FN: " + this.firstName + ", LN: " + this.lastName;
        }
    }

    static class CustomerService {
        private final Map<Integer, Customer> store;

        public CustomerService(int maxId) {
            this.store = new ConcurrentHashMap<>(maxId);
//            this.store = Collections.synchronizedMap(new HashMap<>(maxId));
        }

        public Future<Long> readCustomer(Integer id) {
            final long startTime = System.nanoTime();
            return executor.submit(() -> {
                Customer customer = store.get(id);
//                System.out.println("customer = " + customer);
                long endTime = System.nanoTime();
                return endTime - startTime;
            });
        }

        public Future<Long> upsert(Integer id, Customer customer) {
            final long startTime = System.nanoTime();
            return executor.submit(() -> {
                store.put(id, customer);
                long endTime = System.nanoTime();
                return endTime - startTime;
            });
        }
    }
}
```

## Rust Output
``` text
Start!! 8
====Start====
For 100 cycles
Total duration (reads, write) = (6, 6)ms
Average (read, write) = (15947.62035988038, 15235.508974171851) ops/s
Test Duration:: 6.25398ms
====End====
For 200 cycles
Total duration (reads, write) = (13, 9)ms
Average (read, write) = (15284.600016537937, 21318.652456505686) ops/s
Test Duration:: 12.942747ms
====End====
For 500 cycles
Total duration (reads, write) = (53, 29)ms
Average (read, write) = (9422.532376810612, 17056.49339168337) ops/s
Test Duration:: 43.102057ms
====End====
For 1000 cycles
Total duration (reads, write) = (129, 80)ms
Average (read, write) = (7723.710039628965, 12464.795211454468) ops/s
Test Duration:: 104.703309ms
====End====
For 2000 cycles
Total duration (reads, write) = (385, 220)ms
Average (read, write) = (5193.148971455561, 9053.962995656488) ops/s
Test Duration:: 291.828017ms
====End====
For 3000 cycles
Total duration (reads, write) = (1503, 422)ms
Average (read, write) = (1995.6270330383877, 7102.223159673592) ops/s
Test Duration:: 630.924506ms
====End====
For 4000 cycles
Total duration (reads, write) = (1251, 621)ms
Average (read, write) = (3196.9214950346786, 6440.262242261892) ops/s
Test Duration:: 876.613433ms
====End====
For 6000 cycles
Total duration (reads, write) = (3000, 1270)ms
Average (read, write) = (1999.9095894205543, 4724.052559667057) ops/s
Test Duration:: 2.011311095s
====End====
For 7000 cycles
Total duration (reads, write) = (3704, 1665)ms
Average (read, write) = (1889.811207198967, 4203.264659688774) ops/s
Test Duration:: 2.479233642s
====End====
For 8000 cycles
Total duration (reads, write) = (4385, 2070)ms
Average (read, write) = (1824.3299252945424, 3863.143054096651) ops/s
Test Duration:: 2.970084453s
====End====
For 9000 cycles
Total duration (reads, write) = (5700, 2469)ms
Average (read, write) = (1578.8320726855104, 3644.560518027627) ops/s
Test Duration:: 3.753896872s
====End====
For 10000 cycles
Total duration (reads, write) = (7003, 3056)ms
Average (read, write) = (1427.94627063426, 3271.614997982845) ops/s
Test Duration:: 4.585594433s
====End====
```

## Java Output 
``` text
===Start===
For 100 cycles
Total duration (read, write) = (4.42, 12.47)ms 
Average (read, write) = (22611.231, 8018.119) ops/s
Test Duration:: 66.672ms 
===End===

===Start===
For 200 cycles
Total duration (read, write) = (6.06, 8.31)ms 
Average (read, write) = (33028.191, 24065.260) ops/s
Test Duration:: 6.454ms 
===End===

===Start===
For 500 cycles
Total duration (read, write) = (9.25, 18.24)ms 
Average (read, write) = (54026.235, 27413.602) ops/s
Test Duration:: 12.722ms 
===End===

===Start===
For 1000 cycles
Total duration (read, write) = (256.37, 44.96)ms 
Average (read, write) = (3900.640, 22240.238) ops/s
Test Duration:: 32.877ms 
===End===

===Start===
For 2000 cycles
Total duration (read, write) = (18.75, 22.02)ms 
Average (read, write) = (106674.444, 90840.952) ops/s
Test Duration:: 89.079ms 
===End===

===Start===
For 3000 cycles
Total duration (read, write) = (28.89, 30.19)ms 
Average (read, write) = (103858.812, 99384.607) ops/s
Test Duration:: 55.003ms 
===End===

===Start===
For 4000 cycles
Total duration (read, write) = (46.01, 45.84)ms 
Average (read, write) = (86936.511, 87258.390) ops/s
Test Duration:: 52.976ms 
===End===

===Start===
For 5000 cycles
Total duration (read, write) = (60.37, 62.78)ms 
Average (read, write) = (82820.404, 79640.466) ops/s
Test Duration:: 57.600ms 
===End===

===Start===
For 6000 cycles
Total duration (read, write) = (112.10, 137.65)ms 
Average (read, write) = (53524.938, 43589.713) ops/s
Test Duration:: 73.873ms 
===End===

===Start===
For 7000 cycles
Total duration (read, write) = (112.43, 130.78)ms 
Average (read, write) = (62263.205, 53523.740) ops/s
Test Duration:: 76.347ms 
===End===

===Start===
For 8000 cycles
Total duration (read, write) = (126.98, 80.37)ms 
Average (read, write) = (63003.402, 99541.203) ops/s
Test Duration:: 102.119ms 
===End===

===Start===
For 9000 cycles
Total duration (read, write) = (187.83, 609.90)ms 
Average (read, write) = (47916.879, 14756.422) ops/s
Test Duration:: 129.408ms 
===End===

===Start===
For 10000 cycles
Total duration (read, write) = (92.75, 93.67)ms 
Average (read, write) = (107816.411, 106758.329) ops/s
Test Duration:: 108.464ms 
===End===
```

