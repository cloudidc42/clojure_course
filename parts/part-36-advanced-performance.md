# Part 36: Advanced Performance Engineering
## ขั้นตอนที่ 1051-1080: Profiling, Memory, GC Tuning, Benchmarking, JIT

---

## บทนำ

Performance engineering สำหรับ Clojure applications:
- **Profiling** - ค้นหา bottlenecks จริงๆ
- **Memory management** - heap, GC, off-heap
- **JVM tuning** - GC flags, JIT optimization
- **Criterium** - micro-benchmarking ที่ accurate
- **Async I/O** - ไม่ block threads

---

## ขั้นตอนที่ 1051: Profiling ด้วย JVM Profilers

```clojure
;; Tools:
;; - VisualVM (free, bundled with JDK)
;; - JProfiler (commercial)
;; - YourKit (commercial)
;; - async-profiler (free, flame graphs)

;; Enable JMX for remote profiling
;; JVM flags:
;; -Dcom.sun.management.jmxremote
;; -Dcom.sun.management.jmxremote.port=9090
;; -Dcom.sun.management.jmxremote.authenticate=false

;; async-profiler integration
;; deps.edn
;; {:deps {com.github.jvm-profiling-tools/ap-loader {:mvn/version "2.1.0"}}}

;; Profile specific code section
(defn profile-block [f duration-seconds output-file]
  (let [profiler (one.profiler.AsyncProfiler/getInstance)]
    (.start profiler (str "start,file=" output-file))
    (f)
    (Thread/sleep (* duration-seconds 1000))
    (.stop profiler)))

;; Flame graph interpretation
;; Wide bars = hot code paths
;; Look for: DB calls, JSON parsing, reflection warnings

;; Simple timing helper
(defmacro timed [label & body]
  `(let [start#  (System/nanoTime)
          result# (do ~@body)
          ms#     (/ (- (System/nanoTime) start#) 1e6)]
     (println (format "[%s] %.2f ms" ~label ms#))
     result#))

(timed "database-query"
  (db/find-users-by-criteria ds {:status :active}))
```

---

## ขั้นตอนที่ 1052: Criterium Benchmarking

```clojure
(ns perf.benchmark
  (:require [criterium.core :as crit]))

;; Quick benchmark (less accurate)
(crit/quick-bench (reduce + (range 1000)))

;; Full benchmark (accurate, warm-up JIT)
(crit/bench (reduce + (range 1000)))

;; Compare implementations
(defn sum-loop [n]
  (loop [i 0 total 0]
    (if (> i n)
      total
      (recur (inc i) (+ total i)))))

(defn sum-reduce [n]
  (reduce + (range n)))

(defn sum-formula [n]
  (/ (* n (inc n)) 2))

;; Benchmark all three
(println "Loop:")
(crit/quick-bench (sum-loop 10000))

(println "Reduce:")
(crit/quick-bench (sum-reduce 10000))

(println "Formula:")
(crit/quick-bench (sum-formula 10000))

;; Results:
;; Loop:     ~0.5 ms (without type hints)
;; Reduce:   ~2.0 ms (lazy seq overhead)
;; Formula:  ~5 ns  (O(1)!)

;; With type hints:
(defn sum-loop-typed [^long n]
  (loop [i (long 0) total (long 0)]
    (if (> i n)
      total
      (recur (unchecked-inc i) (unchecked-add total i)))))
;; => ~50 ns (10x faster than without hints)
```

---

## ขั้นตอนที่ 1053: Memory Profiling

```clojure
;; Memory analysis tools
;; - jmap: dump heap
;; - jhat: analyze heap dump
;; - Eclipse MAT: visual heap analysis
;; - VisualVM memory sampler

;; Check memory usage in code
(defn memory-stats []
  (let [runtime  (Runtime/getRuntime)
        mb       (fn [bytes] (/ bytes 1024 1024))]
    {:total-mb     (mb (.totalMemory runtime))
     :free-mb      (mb (.freeMemory runtime))
     :used-mb      (mb (- (.totalMemory runtime) (.freeMemory runtime)))
     :max-mb       (mb (.maxMemory runtime))}))

;; Force GC (for testing, not production)
(System/gc)

;; Watch for memory leaks
;; Common causes in Clojure:
;; 1. Lazy sequences held in vars (head retention)
;; 2. Infinite sequences not consumed lazily
;; 3. Atoms holding too much state

;; Head retention example (memory leak!)
(defn bad-process-large-file [file]
  (let [lines (line-seq (clojure.java.io/reader file))]
    ;; PROBLEM: holding head of 'lines' while also consuming
    (println "Total lines:" (count lines))  ; realizes all lines
    (println "First line:" (first lines))   ; head retention!
    ))

;; Fix: don't hold head
(defn good-process-large-file [file]
  (with-open [reader (clojure.java.io/reader file)]
    (->> (line-seq reader)
         (reduce (fn [{:keys [count first-line]} line]
                    {:count      (inc count)
                     :first-line (or first-line line)})
                  {:count 0 :first-line nil}))))
```

---

## ขั้นตอนที่ 1054: JVM GC Tuning

```bash
# ===== GC Algorithm Selection =====

# G1GC (default Java 9+, balanced latency/throughput)
-XX:+UseG1GC

# ZGC (low latency, Java 15+, sub-ms pauses)
-XX:+UseZGC

# Shenandoah (low latency, RedHat OpenJDK)
-XX:+UseShenandoahGC

# ParallelGC (max throughput, not latency-sensitive)
-XX:+UseParallelGC

# ===== Heap Sizing =====
-Xms512m         # Min heap (set = max to avoid GC on resize)
-Xmx2g           # Max heap
-XX:MaxMetaspaceSize=256m  # Metaspace (class definitions)

# For container environments:
-XX:MaxRAMPercentage=75.0  # Use 75% of container memory

# ===== G1GC Tuning =====
-XX:MaxGCPauseMillis=200   # Target max pause (G1 tries to meet this)
-XX:G1HeapRegionSize=16m   # Region size (1-32MB, power of 2)

# ===== Logging =====
-Xlog:gc*:file=gc.log:time,uptime,level,tags  # Java 9+
-verbose:gc  # Simple GC logging

# ===== JIT =====
-server             # Server JIT (more aggressive optimization)
-XX:+OptimizeStringConcat   # String concat optimization
-XX:+UseCompressedOops      # 32-bit object pointers (saves memory)
```

---

## ขั้นตอนที่ 1055: Optimizing Hot Paths

```clojure
;; Identify and optimize hot paths

;; 1. Avoid object allocation in loops
;; BAD: creates StringBuilder every iteration
(defn join-strings-bad [strings]
  (reduce #(str %1 "," %2) strings))

;; GOOD: single StringBuilder
(defn join-strings-good [strings]
  (let [sb (StringBuilder.)]
    (loop [[s & more] strings]
      (when s
        (.append sb s)
        (when more (.append sb ","))
        (recur more)))
    (.toString sb)))

;; Or simply:
(clojure.string/join "," strings)

;; 2. Use transients for local mutations
(defn build-map-fast [pairs]
  (persistent!
    (reduce (fn [m [k v]]
              (assoc! m k v))
            (transient {})
            pairs)))

;; 3. Avoid seqs for tight loops
;; BAD: lazy seq overhead
(defn sum-lazy [coll]
  (reduce + coll))

;; GOOD: arrays + direct access
(defn sum-array-fast [^doubles arr]
  (let [n (alength arr)]
    (loop [i 0 total 0.0]
      (if (>= i n)
        total
        (recur (unchecked-inc i)
               (unchecked-add total (aget arr i)))))))

;; 4. Memoize expensive computations
(def expensive-fn-memo
  (memoize expensive-fn))

;; 5. Use protocols instead of multimethods for performance
;; Multimethods: runtime dispatch via hashing (slower)
;; Protocols: JVM interface dispatch (faster)
```

---

## ขั้นตอนที่ 1056: Non-blocking I/O

```clojure
;; Async HTTP client สำหรับ high-throughput
(ns perf.async-http
  (:require [org.httpkit.client :as http-client]
            [clojure.core.async :as async]))

;; Parallel HTTP requests (non-blocking)
(defn fetch-all-parallel [urls]
  (let [promises (mapv (fn [url]
                          (let [p (promise)]
                            (http-client/get url
                              {:timeout 5000}
                              (fn [resp]
                                (deliver p resp)))
                            p))
                        urls)]
    (mapv deref promises)))

;; With core.async
(defn fetch-with-channel [url]
  (let [ch (async/chan 1)]
    (http-client/get url
      {:timeout 5000}
      (fn [resp]
        (async/go (async/>! ch resp))))
    ch))

(defn fetch-all-async [urls]
  (async/<!! 
    (async/map vector
      (map fetch-with-channel urls))))

;; Connection pooling for HTTP clients
(def http-pool
  {:timeout           10000
   :pool              {:threads         10
                       :max-queued-calls 100}
   :keepalive         30000})
```

---

## Project: Performance Test Suite

```clojure
(ns perf.suite
  (:require [criterium.core :as crit]))

(defn run-benchmarks! []
  (println "=== Performance Benchmarks ===\n")
  
  (println "1. Database query (uncached):")
  (crit/quick-bench (db/find-user 1))
  
  (println "\n2. Database query (cached):")
  (crit/quick-bench (cache/get-or-fetch "user:1" #(db/find-user 1)))
  
  (println "\n3. JSON serialization (500 items):")
  (let [data (repeat 500 {:id 1 :name "test" :value 42})]
    (crit/quick-bench (cheshire.core/generate-string data)))
  
  (println "\n4. Parallel vs sequential (10 HTTP calls):")
  (let [urls (repeat 10 "http://httpbin.org/get")]
    (println "  Sequential:")
    (crit/quick-bench (mapv #(http/get %) urls))
    (println "  Parallel:")
    (crit/quick-bench (fetch-all-parallel urls)))
  
  (println "\n=== Memory Stats ===")
  (println (memory-stats)))
```

---

*Part 36 จาก 100+ | ขั้นตอน 1051-1080 จาก 1000+*
