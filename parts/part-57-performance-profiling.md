# Part 57: Performance Profiling
## ขั้นตอนที่ 1681-1710: JVM Profiling, Memory Optimization, Benchmarking, Lazy Sequences

---

## บทนำ

Performance optimization สำหรับ Clojure:
- **JVM profiling** - async-profiler, YourKit
- **Memory analysis** - heap dumps, GC tuning
- **Benchmarking** - criterium, JMH
- **Lazy sequences** - หลีกเลี่ยง memory leaks
- **Transients** - mutation สำหรับ speed

---

## ขั้นตอนที่ 1681: Benchmarking ด้วย Criterium

```clojure
(ns myapp.benchmark
  (:require [criterium.core :as crit]))

;; Quick benchmark
(crit/quick-bench (+ 1 2 3))

;; Full benchmark (more accurate)
(crit/bench (reduce + (range 1000)))

;; Compare implementations
(defn sum-loop [n]
  (loop [i 0 acc 0]
    (if (= i n) acc
      (recur (inc i) (+ acc i)))))

(defn sum-reduce [n]
  (reduce + (range n)))

(defn sum-formula [n]
  (/ (* n (dec n)) 2))

;; Benchmark all three
(println "Loop:")
(crit/quick-bench (sum-loop 10000))

(println "Reduce:")
(crit/quick-bench (sum-reduce 10000))

(println "Formula:")
(crit/quick-bench (sum-formula 10000))

;; Benchmark with warm-up
(defn benchmark-with-stats [f label]
  (println "\n=== " label " ===")
  (let [result (crit/benchmark (f) {})]
    (println "Mean:" (-> result :mean first (* 1e9) int) "ns")
    (println "Std:" (-> result :variance first Math/sqrt (* 1e9) int) "ns")
    (println "Min:" (-> result :lower-q first (* 1e9) int) "ns")
    result))
```

---

## ขั้นตอนที่ 1682: Transients สำหรับ Performance

```clojure
(ns myapp.transients)

;; Transients: mutable version of persistent data structures
;; Use when building large collections in a single-threaded context

;; Building large map: persistent (slow)
(defn build-map-persistent [n]
  (reduce (fn [m i] (assoc m i (* i i)))
           {}
           (range n)))

;; Building large map: transient (fast ~3x)
(defn build-map-transient [n]
  (persistent!
    (reduce (fn [m i] (assoc! m i (* i i)))
             (transient {})
             (range n))))

;; Building large vector
(defn build-vec-transient [n]
  (persistent!
    (reduce (fn [v i] (conj! v i))
             (transient [])
             (range n))))

;; Group-by with transient
(defn group-by-fast [f coll]
  (persistent!
    (reduce (fn [m x]
               (let [k (f x)]
                 (assoc! m k (conj (get m k []) x))))
             (transient {})
             coll)))

;; Frequency count with transient
(defn frequencies-fast [coll]
  (persistent!
    (reduce (fn [m x]
               (assoc! m x (inc (get m x 0))))
             (transient {})
             coll)))

;; Benchmark comparison
;; (crit/quick-bench (build-map-persistent 100000))
;; (crit/quick-bench (build-map-transient 100000))
```

---

## ขั้นตอนที่ 1683: Lazy Sequences คอยระวัง

```clojure
(ns myapp.lazy-sequences)

;; Lazy seq: computed on demand, not all at once
(defn lazy-integers []
  (iterate inc 0))  ; infinite sequence!

;; Safe: take only what you need
(take 10 (lazy-integers))  ; => (0 1 2 3 4 5 6 7 8 9)

;; DANGER: holding onto head prevents GC
;; This keeps entire sequence in memory!
(let [all-nums (range 1000000)]
  (println "First:" (first all-nums))
  ;; all-nums still referenced here - can't be GC'd
  (println "Count:" (count all-nums)))

;; SAFE: don't hold head
(let [sum (reduce + (range 1000000))]  ; range is lazy, reduce consumes it
  (println "Sum:" sum))

;; Chunked seqs: lazy but in 32-element chunks
;; Sometimes causes surprising behavior
(defn check-chunking []
  (let [result (map (fn [x]
                      (println "Processing" x)
                      (* x x))
                     (range 100))]
    (first result)))  ; prints 32 items, not 1!

;; Use eduction or lazy-seq for true laziness
(defn truly-lazy [coll f]
  (lazy-seq
    (when-let [s (seq coll)]
      (cons (f (first s))
            (truly-lazy (rest s) f)))))

;; Sequence pipeline optimization
;; BAD: creates intermediate collections
(defn process-bad [data]
  (->> data
       (filter :active?)
       (map transform)
       (take 100)))

;; GOOD: transducer, no intermediate collections
(defn process-good [data]
  (into []
    (comp
      (filter :active?)
      (map transform)
      (take 100))
    data))
```

---

## ขั้นตอนที่ 1684: Memory Optimization

```clojure
(ns myapp.memory)

;; Reduce object creation
;; BAD: creates many intermediate strings
(defn build-sql-bad [parts]
  (reduce str parts))  ; O(n²) due to string concatenation

;; GOOD: StringBuilder
(defn build-sql-good [parts]
  (let [sb (StringBuilder.)]
    (doseq [p parts]
      (.append sb p))
    (.toString sb)))

;; Or use clojure.string/join
(defn build-sql-best [parts]
  (clojure.string/join "" parts))

;; Avoid boxing with type hints
(defn sum-typed
  "Sum longs without boxing"
  ^long [^longs arr]
  (loop [i   0
         sum 0]
    (if (= i (alength arr))
      sum
      (recur (unchecked-inc i)
             (unchecked-add sum (aget arr i))))))

;; Weak references for caches
(defn create-weak-cache []
  (java.util.WeakHashMap.))

;; SoftReference for memory-sensitive cache
(defn soft-cache []
  (let [cache (java.util.concurrent.ConcurrentHashMap.)]
    {:put! (fn [k v]
             (.put cache k (java.lang.ref.SoftReference. v)))
     :get  (fn [k]
             (when-let [ref (.get cache k)]
               (.get ref)))}))

;; Monitor GC pressure
(defn gc-stats []
  (let [beans (java.lang.management.ManagementFactory/getGarbageCollectorMXBeans)]
    (map (fn [bean]
           {:name       (.getName bean)
            :count      (.getCollectionCount bean)
            :time-ms    (.getCollectionTime bean)})
         beans)))
```

---

## ขั้นตอนที่ 1685: JVM Profiling Setup

```clojure
;; async-profiler: low-overhead CPU profiler
;; deps.edn: {:deps {com.clojure-goes-fast/clj-async-profiler {:mvn/version "1.2.2"}}}

(ns myapp.profiling
  (:require [clj-async-profiler.core :as prof]))

;; Profile code
(prof/profile
  (dotimes [_ 1000]
    (my-expensive-function)))

;; Save flamegraph
(prof/profile-for 10  ; seconds
  {:output-file "/tmp/profile.html"})

;; View flamegraph at http://localhost:port

;; JVM startup flags for profiling
;; -XX:+UnlockDiagnosticVMOptions
;; -XX:+DebugNonSafepoints
;; These allow async-profiler to capture accurate stack traces

;; Memory analysis
(defn heap-info []
  (let [runtime (Runtime/getRuntime)]
    {:total-memory (/ (.totalMemory runtime) 1e6)
     :free-memory  (/ (.freeMemory  runtime) 1e6)
     :used-memory  (/ (- (.totalMemory runtime) (.freeMemory runtime)) 1e6)
     :max-memory   (/ (.maxMemory   runtime) 1e6)}))

;; Force GC (for testing)
(defn force-gc! []
  (System/gc)
  (Thread/sleep 100))

;; Measure memory allocation
(defmacro measure-allocation [& body]
  `(let [before# (.getUsed (.getHeapMemoryUsage
                              (java.lang.management.ManagementFactory/getMemoryMXBean)))
          result# (do ~@body)
          after#  (.getUsed (.getHeapMemoryUsage
                               (java.lang.management.ManagementFactory/getMemoryMXBean)))]
     (println "Allocated:" (/ (- after# before#) 1024.0) "KB")
     result#))
```

---

## Project: Performance-Critical Data Processor

```clojure
(ns data.fast-processor
  (:require [criterium.core :as crit]))

;; Optimized batch processor
(defn process-batch
  "Process large batch with minimal allocation"
  [^java.util.List items transform-fn filter-fn]
  (let [n      (count items)
        result (java.util.ArrayList. n)]
    (dotimes [i n]
      (let [item      (.get items i)
            transformed (transform-fn item)]
        (when (filter-fn transformed)
          (.add result transformed))))
    (vec result)))

;; Parallel processing with controlled parallelism
(defn pmap-bounded [f coll parallelism]
  (let [batches (partition-all
                  (max 1 (quot (count coll) parallelism))
                  coll)]
    (mapcat #(map f %) (pmap #(doall (map f %)) batches))))

;; Statistics pipeline - optimized
(defn fast-stats [numbers]
  (let [arr    (double-array numbers)
        n      (alength arr)
        _      (java.util.Arrays/sort arr)
        sum    (areduce arr i acc 0.0 (+ acc (aget arr i)))
        mean   (/ sum n)
        median (if (odd? n)
                 (aget arr (quot n 2))
                 (/ (+ (aget arr (dec (quot n 2)))
                       (aget arr (quot n 2)))
                    2.0))]
    {:n      n
     :sum    sum
     :mean   mean
     :median median
     :min    (aget arr 0)
     :max    (aget arr (dec n))
     :p95    (aget arr (int (* n 0.95)))
     :p99    (aget arr (int (* n 0.99)))}))
```

---

*Part 57 จาก 100+ | ขั้นตอน 1681-1710 จาก 1000+*
