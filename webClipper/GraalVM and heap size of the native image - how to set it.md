# GraalVM and heap size of the native image - how to set it?
In the following example, I will use the native image I’ve [created for my demo Ratpack](https://e.printstacktrace.blog/ratpack-graalvm-how-to-start/ "created for my demo Ratpack") application. The only difference is that this time I have configured this application to use a single thread for handling event loop events. _(HTTP requests in particular.)_

| Note | In the following example I also use `-XX:+PrintGC`, `-XX:+PrintGCTimeStamps`, `-XX:+VerboseGC` and `+XX:+PrintHeapShape` flags to log verbose garbage collector activity to the console. |

For testing purpose, I’m going to execute **200** parallel requests, repeated **100** times, so our application will have to handle **20,000** requests in total. I use `siege` \[[1](#_footnotedef_1 "View footnote.")] tool to execute those requests. Firstly, let’s do this test with the application executed without specifying the heap size. However, what is the **default maximum heap size**? You can find it out using GraalVM polyglot launcher for instance:

Listing 2. Finding the default maximum heap size

````null
$ polyglot --native --help:vm | grep HeapSize
  --vm.XX:MaximumHeapSizePercent=80            The maximum heap size as percent of physical memory.
  --vm.Xmx<value>                              Sets the maximum size of the heap, in bytes. Default: MaximumHeapSizePercent * physical memory.```

As you can see, it’s set to **80%** of the physical memory. In my case it is something around **12.29 GB**. Let’s run the application.

Listing 3. Running Ratpack application with the default max heap size

```null
$ ./ratpack-graalvm-demo -XX:+PrintGC -XX:+PrintGCTimeStamps -XX:+VerboseGC +XX:+PrintHeapShape
[main] INFO ratpack.server.RatpackServer - Starting server...
[main] INFO ratpack.server.RatpackServer - Building registry...
[main] INFO ratpack.server.RatpackServer - Ratpack started for http://localhost:5050```

We can execute **20,000** requests now.

Listing 4. Executing 20k requests using `siege`

```null
$ siege -c 200 -r 100 http://localhost:5050/
...
HTTP/1.1 200     0.01 secs:      53 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:      53 bytes ==> GET  /
HTTP/1.1 200     0.00 secs:      53 bytes ==> GET  /

Transactions:		       20000 hits
Availability:		      100.00 %
Elapsed time:		        4.31 secs
Data transferred:	        1.01 MB
Response time:		        0.03 secs
Transaction rate:	     4640.37 trans/sec
Throughput:		        0.23 MB/sec
Concurrency:		      143.19
Successful transactions:       20000
Failed transactions:	           0
Longest transaction:	        3.14
Shortest transaction:	        0.02```

Let’s take a look at the console where the application is running.

```null
$ ./ratpack-graalvm-demo -XX:+PrintGC -XX:+PrintGCTimeStamps -XX:+VerboseGC +XX:+PrintHeapShape
[main] INFO ratpack.server.RatpackServer - Starting server...
[main] INFO ratpack.server.RatpackServer - Building registry...
[main] INFO ratpack.server.RatpackServer - Ratpack started for http://localhost:5050
[Heap policy parameters:
  YoungGenerationSize: 268435456
      MaximumHeapSize: 13193507600
      MinimumHeapSize: 536870912
     AlignedChunkSize: 1048576
  LargeArrayThreshold: 131072]
[[36323 msec:  GC: before  epoch: 1  cause: CollectOnAllocation.Sometimes]
[36333 msec: Incremental GC (CollectOnAllocation.Sometimes) 261172K->17403K, 0.0105428 secs]
 [36333 msec:  GC: after   epoch: 1  cause: CollectOnAllocation.Sometimes  policy: by time: 50% in incremental collections  type: incremental
  collection time: 10542877 nanoSeconds]]```

As you can see, there was no much activity from the garbage collector. Here what does the memory consumption of this process looks like:

```null
$ ps ef -o command,vsize,rss,%mem,size | grep ratpack | head -n 1
 \_ ./ratpack-graalvm-demo  525924 280148  1.7 307328```

It consumed **280 MB** RSS. Is it much? Maybe. Let’s run the same application with `-Xmx64m` and check if it makes any difference.

Listing 5. Running Ratpack application with max heap size set to 64 MB

```null
$ ./ratpack-graalvm-demo -XX:+PrintGC -XX:+PrintGCTimeStamps -XX:+VerboseGC +XX:+PrintHeapShape -Xmx64m
[main] INFO ratpack.server.RatpackServer - Starting server...
[main] INFO ratpack.server.RatpackServer - Building registry...
[main] INFO ratpack.server.RatpackServer - Ratpack started for http://localhost:5050```

Application started. It’s time to execute the same amount of requests.

Listing 6. Executing 20k requests using `siege`

```null
$ siege -c 200 -r 100 http://localhost:5050/
...
HTTP/1.1 200     0.01 secs:      53 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:      53 bytes ==> GET  /
HTTP/1.1 200     0.00 secs:      53 bytes ==> GET  /

Transactions:		       20000 hits
Availability:		      100.00 %
Elapsed time:		        3.86 secs
Data transferred:	        1.01 MB
Response time:		        0.03 secs
Transaction rate:	     5181.35 trans/sec
Throughput:		        0.26 MB/sec
Concurrency:		      152.01
Successful transactions:       20000
Failed transactions:	           0
Longest transaction:	        3.09
Shortest transaction:	        0.01```

This time the garbage collector was much more active - here is the console log generated while the application was handling 20k requests:

```null
$ ./ratpack-graalvm-demo -XX:+PrintGC -XX:+PrintGCTimeStamps -XX:+VerboseGC +XX:+PrintHeapShape -Xmx64m
[main] INFO ratpack.server.RatpackServer - Starting server...
[main] INFO ratpack.server.RatpackServer - Building registry...
[main] INFO ratpack.server.RatpackServer - Ratpack started for http://localhost:5050
[Heap policy parameters:
  YoungGenerationSize: 6710880
      MaximumHeapSize: 67108864
      MinimumHeapSize: 13421760
     AlignedChunkSize: 1048576
  LargeArrayThreshold: 131072]
[[1768 msec:  GC: before  epoch: 1  cause: CollectOnAllocation.Sometimes]
[1772 msec: Incremental GC (CollectOnAllocation.Sometimes) 20463K->17403K, 0.0040444 secs]
 [1772 msec:  GC: after   epoch: 1  cause: CollectOnAllocation.Sometimes  policy: by time: 50% in incremental collections  type: incremental
  collection time: 4044472 nanoSeconds]]
[[1845 msec:  GC: before  epoch: 2  cause: CollectOnAllocation.Sometimes]
[1850 msec: Full GC (CollectOnAllocation.Sometimes) 24543K->17403K, 0.0049823 secs]
 [1850 msec:  GC: after   epoch: 2  cause: CollectOnAllocation.Sometimes  policy: by time: 50% in incremental collections  type: complete
  collection time: 4982361 nanoSeconds]]



[[5479 msec:  GC: before  epoch: 53  cause: CollectOnAllocation.Sometimes]
[5483 msec: Full GC (CollectOnAllocation.Sometimes) 24543K->17403K, 0.0042101 secs]
 [5483 msec:  GC: after   epoch: 53  cause: CollectOnAllocation.Sometimes  policy: by time: 50% in incremental collections  type: complete
  collection time: 4210195 nanoSeconds]]
[[5549 msec:  GC: before  epoch: 54  cause: CollectOnAllocation.Sometimes]
[5551 msec: Incremental GC (CollectOnAllocation.Sometimes) 24543K->17403K, 0.0022523 secs]
 [5551 msec:  GC: after   epoch: 54  cause: CollectOnAllocation.Sometimes  policy: by time: 50% in incremental collections  type: incremental
  collection time: 2252302 nanoSeconds]]```

And now let’s take a look at the memory consumption.

```null
$ ps ef -o command,vsize,rss,%mem,size | grep ratpack | head -n 1
 \_ ./ratpack-graalvm-demo  281188 35812  0.2 62592```

We could expect that. The application run with much smaller maximum heap size consumed **eight times** less memory - **35 MB** in this case. 
 [https://e.printstacktrace.blog/graalvm-heap-size-of-native-image-how-to-set-it/](https://e.printstacktrace.blog/graalvm-heap-size-of-native-image-how-to-set-it/)
````
