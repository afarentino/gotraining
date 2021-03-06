## Memory Tracing

There is no way to identify specifically in the code where a leak is occuring. We can validate if a memory leak is present and which functions or methods are producing the most allocations.

## GODEBUG

[http://golang.org/pkg/runtime/](http://golang.org/pkg/runtime/)  
[http://dave.cheney.net/2014/07/11/visualising-the-go-garbage-collector](Visualising the Go garbage collector)  
[http://dave.cheney.net/2015/11/29/a-whirlwind-tour-of-gos-runtime-environment-variables](Tour of Go's env variables)  
[https://deferpanic.com/blog/understanding-golang-memory-usage](Understanding Go memory usage)

To validate if a memory leak is truly occuring use the GODEBUG environmental variable. Setting gctrace=1 causes the garbage collector to emit a single line to standard error at each collection, summarizing the amount of memory collected and the length of the pause. Setting gctrace=2 emits the same summary but also repeats each collection. The format of this line is subject to change:

    export GODEBUG=gctrace=1

    gc # @#s #%: #+...+# ms clock, #+...+# ms cpu, #->#-># MB, # MB goal, # P

Where the fields are as follows:

    gc #        the GC number, incremented at each GC
    @#s         time in seconds since program start
    #%          percentage of time spent in GC since program start
    #+...+#     wall-clock/CPU times for the phases of the GC
    #->#-># MB  heap size at GC start, at GC end, and live heap
    # MB goal   goal heap size
    # P         number of processors used

In C++, a memory leak is memory you have lost a reference to.
In Go, a memory leak is memory you retain a reference to.

### Running a GODEBUG Trace

    go build
    GODEBUG=gctrace=1 ./memory_trace

    gc 1 @0.009s 1%: 0.059+0.17+0.005+0.24+0.12 ms clock, 0.17+0.17+0+0/0.36/0.067+0.38 ms cpu, 5->5->3 MB, 4 MB goal, 8 P
    gc 2 @0.017s 1%: 0.037+0.096+0.098+0.21+0.086 ms clock, 0.22+0.096+0+0.10/0.31/0.091+0.51 ms cpu, 8->8->7 MB, 7 MB goal, 8 P
    gc 3 @0.032s 1%: 0.020+0.16+0.007+0.25+0.090 ms clock, 0.14+0.16+0+0/0.20/0.27+0.63 ms cpu, 17->17->14 MB, 14 MB goal, 8 P
    gc 4 @0.066s 0%: 0.029+0.17+0.074+0.48+0.10 ms clock, 0.20+0.17+0+0/0.42/0.26+0.76 ms cpu, 35->35->29 MB, 29 MB goal, 8 P

## pprof heap

We can get detailed information about the heap using the pprof support. We can actually produce and compare different profiles to see differences in memory over time.

### Comparing Profiles

Build and run the service:
```
    go build
    ./memory_trace
```

Take a snapshot of the current heap profile:
```
    curl -s http://localhost:6060/debug/pprof/heap > base.heap
```

After some time, take another snapshot:
```
    curl -s http://localhost:6060/debug/pprof/heap > current.heap
```

Now compare both snapshots against the binary and get into the pprof tool:
```
    go tool pprof -alloc_space -base base.heap memory_trace current.heap

    -inuse_space  : Display in-use memory size
    -inuse_objects: Display in-use object counts
    -alloc_space  : Display allocated memory size
    -alloc_objects: Display allocated object counts
```

### Running pprof Commands

Run the `top` command to see the functions allocating the most objects:
```
    (pprof) top
        2.24GB of 2.24GB total (  100%)
              flat  flat%   sum%        cum   cum%
            2.24GB   100%   100%     2.24GB   100%  main.main.func1
                 0     0%   100%     2.24GB   100%  runtime.goexit
```

Run the `list` command against the goroutine declared in main:
```
    (pprof) list main
        Total: 2.24GB
        ROUTINE ======================== main.main.func1 in /Users/bill/code/go/src/github.com/ardanlabs/gotraining/topics/memory_trace/trace.go
            2.24GB     2.24GB (flat, cum)   100% of Total
                 .          .     17:   // to be constantly shuffled around, this becomes very expensive.
                 .          .     18:   go func() {
                 .          .     19:       m := make(map[int]int)
                 .          .     20:
                 .          .     21:       for i := 0; ; i++ {
            2.24GB     2.24GB     22:           m[i] = i
                 .          .     23:       }
                 .          .     24:   }()
                 .          .     25:
                 .          .     26:   // Start a listener for the pprof support.
                 .          .     27:   go func() {
```

### Generate PDF Call Graph

You can generate a call graph but you need Graphviz and Ghostscript to generate a PDF. Look at [Profiling](../profiling).
```
    go tool pprof -pdf -alloc_space -base base.heap memory_trace current.heap > mem.pdf
    open -a "Adobe Acrobat" mem.pdf
```

## Links

https://golang.org/pkg/runtime  
https://www.hakkalabs.co/articles/finding-memory-leaks-go-programs

## Code Review

[Finding Leak](trace.go) ([Go Playground](http://play.golang.org/p/W2zH_cR5Ir))
___
All material is licensed under the [Apache License Version 2.0, January 2004](http://www.apache.org/licenses/LICENSE-2.0).
