# Part 15: Performance Optimization
## ขั้นตอนที่ 421-450: Profiling, JVM Tuning, Benchmarking

---

## บทนำ

Clojure ทำงานบน JVM ซึ่งมี performance ดีมาก แต่มี overhead บ้าง:
- Persistent data structures (เร็วกว่าที่คิด!)
- Lazy sequences (อาจ overhead สำหรับ small collections)
- Dynamic dispatch
- Boxing/unboxing สำหรับ primitives

การ optimize ที่ถูกต้อง: **Measure first, optimize second!**

---

## ขั้นตอนที่ 421: Benchmarking ด้วย Criterium

```clojure
;; deps.edn
;; {:deps {criterium/criterium {:mvn/version "0.4.6"}}}

(require '[criterium.core :as crit])

;; Quick benchmark
(crit/quick-bench (+ 1 2))

;; Full benchmark (more accurate)
(crit/bench (reduce + (range 1000)))

;; Output:
;; Evaluation count : 60 in 6 samples of 10 calls.
;;              Execution time mean : 15.234 µs
;;     Execution time std-deviation : 0.456 µs
;;    Execution time lower quantile : 14.834 µs ( 2.5%)
;;    Execution time upper quantile : 15.844 µs (97.5%)

;; Compare approaches
(println "== Vector creation ==")
(crit/quick-bench (vec (range 1000)))
(println "== Transient vector ==")
(crit/quick-bench
  (let [v (transient [])]
    (loop [i 0 v v]
      (if (< i 1000)
        (recur (inc i) (conj! v i))
        (persistent! v)))))
```

---

## ขั้นตอนที่ 422: Type Hints สำหรับ Primitive Performance

```clojure
;; Boxing overhead: Clojure แปลง Java primitives เป็น Objects
;; Type hints บอก compiler ใช้ primitives โดยตรง

;; ❌ Slow: boxing/unboxing
(defn slow-sum [n]
  (loop [i 0 sum 0]
    (if (> i n)
      sum
      (recur (inc i) (+ sum i)))))

;; ✅ Fast: primitive arithmetic
(defn fast-sum [^long n]
  (loop [i 0
         sum 0]
    (if (> i n)
      sum
      (recur (unchecked-inc i)          ; unchecked = no overflow check
             (unchecked-add sum i)))))

;; Type hints for arrays
(defn array-sum [^longs arr]
  (let [n (alength arr)]
    (loop [i 0 sum 0]
      (if (< i n)
        (recur (inc i) (+ sum (aget arr i)))
        sum))))

;; Make primitive array
(def my-arr (long-array [1 2 3 4 5]))

;; Check for reflection warnings
(set! *warn-on-reflection* true)
;; Clojure จะ warn เมื่อมี reflection ที่อาจช้า

;; Type hint for method calls
(defn get-length [^String s]
  (.length s))  ; No reflection!
```

---

## ขั้นตอนที่ 423: Transients

```clojure
;; Transients: mutable version of persistent collections
;; ใช้สำหรับ local performance-critical code
;; ต้อง call persistent! เมื่อเสร็จ

;; ❌ Slow: สร้าง intermediate collections
(defn build-map-slow [keys vals]
  (reduce (fn [m [k v]] (assoc m k v))
          {}
          (map vector keys vals)))

;; ✅ Fast: transient
(defn build-map-fast [keys vals]
  (persistent!
    (reduce (fn [m [k v]] (assoc! m k v))
            (transient {})
            (map vector keys vals))))

;; Transient operations:
;; (assoc! transient key val)
;; (dissoc! transient key)
;; (conj! transient val)
;; (pop! transient)
;; (persistent! transient) - convert back

;; Example: counting
(defn count-chars-fast [s]
  (persistent!
    (reduce (fn [counts c]
              (assoc! counts c (inc (get counts c 0))))
            (transient {})
            s)))

;; Benchmark comparison
;; Transients are typically 2-4x faster for bulk operations
```

---

## ขั้นตอนที่ 424: Lazy vs Eager Evaluation

```clojure
;; Lazy sequences: คำนวณ on-demand
;; ดีสำหรับ: large/infinite sequences, short-circuit
;; ไม่ดีสำหรับ: small collections (overhead)

;; Eager ด้วย transducers (ไม่สร้าง intermediate collections)
;; ✅ Efficient: single pass
(def xf (comp (filter even?) (map #(* % %))))
(into [] xf (range 1000))

;; ❌ Less efficient: creates intermediate sequences
(->> (range 1000)
     (filter even?)     ; creates new lazy seq
     (map #(* % %))     ; creates another lazy seq
     (into []))          ; realize both

;; mapv, filterv = eager versions (return vectors)
(mapv inc [1 2 3])     ; eager
(map inc [1 2 3])      ; lazy

;; reduce > loop > for > map+filter for raw speed
(reduce + 0 (range 1000))           ; fastest
(transduce (map inc) + 0 (range 1000)) ; also very fast

;; eduction = composable lazy transducer (better than threading)
(def processed
  (eduction (filter even?) (map #(* % %)) (range 1000)))

(reduce + processed)  ; single pass, no intermediate collection!
```

---

## ขั้นตอนที่ 425: JVM Tuning

```bash
# ===== JVM Flags =====

# Heap size
-Xms512m      # Initial heap (avoid GC on startup)
-Xmx2g        # Maximum heap

# GC Selection
-XX:+UseG1GC              # G1GC (recommended for most apps)
-XX:+UseZGC               # ZGC (low latency, JDK 11+)
-XX:+UseShenandoahGC      # Shenandoah (low pause, JDK 12+)

# G1GC tuning
-XX:MaxGCPauseMillis=200  # Target max GC pause
-XX:G1HeapRegionSize=16m  # Region size

# AOT compilation
-XX:+TieredCompilation    # Enable tiered compilation (default JDK 8+)

# For Clojure specifically
-Dclojure.compile.path=classes
-XX:+OptimizeStringConcat
-XX:+UseCompressedOops    # 32-bit object refs (saves memory)

# Monitoring
-verbose:gc
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps

# JVM startup flags (for faster startup)
-server               # Server JIT compiler
-XX:+AggressiveOpts   # Experimental optimizations
```

```clojure
;; Leiningen JVM flags
;; project.clj
{:jvm-opts ["-Xms512m" "-Xmx2g" 
             "-XX:+UseG1GC"
             "-XX:MaxGCPauseMillis=200"
             "-server"]}

;; deps.edn
;; :aliases {:run {:jvm-opts ["-Xmx2g" "-XX:+UseG1GC"]}}
```

---

## ขั้นตอนที่ 426: Memory Optimization

```clojure
;; ===== Reduce memory allocation =====

;; 1. Use keywords instead of strings for map keys
;; ❌ Uses more memory
(def user {"name" "สมชาย" "age" 25})

;; ✅ Keywords are interned (shared instances)
(def user {:name "สมชาย" :age 25})

;; 2. defrecord vs map (faster field access, less memory)
;; ❌ Generic map
(def point {:x 1.0 :y 2.0})

;; ✅ Record (typed, faster, less overhead)
(defrecord Point [^double x ^double y])
(def p (->Point 1.0 2.0))
(:x p)  ; Fast field access!

;; 3. Avoid object creation in hot loops
;; ❌ Creates map on every iteration
(defn hot-loop-bad [n]
  (reduce (fn [acc i]
            (assoc acc :last i))   ; new map every iteration
          {}
          (range n)))

;; ✅ Use transient or volatile
(defn hot-loop-good [n]
  (loop [i 0 last-val nil]
    (if (< i n)
      (recur (inc i) i)
      last-val)))

;; 4. Cache expensive computations
(def memoized-fib
  (memoize (fn fib [n]
    (if (< n 2) n
        (+ (fib (dec n)) (fib (- n 2)))))))
```

---

## ขั้นตอนที่ 427: Profiling ด้วย VisualVM

```
1. เปิด VisualVM (หรือ JProfiler, YourKit)
2. Attach ไปยัง running Clojure process
3. สร้าง CPU Profiler sampling
4. ดู hotspots (methods ที่ใช้เวลามากที่สุด)

Common Clojure bottlenecks:
- clojure.lang.RT.seq() - การสร้าง lazy sequences
- clojure.core/map/filter - intermediate collections
- toString() - string concatenation
- equals()/hashCode() - collection comparisons

Practical profiling:
```

```clojure
;; Use timbre/System.currentTimeMillis สำหรับ quick profiling
(defmacro profile [label & body]
  `(let [start# (System/nanoTime)
         result# (do ~@body)
         elapsed# (/ (- (System/nanoTime) start#) 1e6)]
     (printf "%s: %.2fms%n" ~label elapsed#)
     result#))

(profile "build-index"
  (build-search-index documents))

;; Thread dump สำหรับ deadlock/performance issues
(.printStackTrace (Thread/currentThread))

;; Memory analysis
(.totalMemory (Runtime/getRuntime))
(.freeMemory  (Runtime/getRuntime))
(.maxMemory   (Runtime/getRuntime))
```

---

## ขั้นตอนที่ 428: AOT Compilation

```clojure
;; AOT = Ahead-of-Time Compilation
;; Precompile Clojure → Java bytecode

;; project.clj
;; :aot :all
;; หรือ specific namespaces:
;; :aot [myapp.core]

;; deps.edn + build.clj
(ns build
  (:require [clojure.tools.build.api :as b]))

(defn compile-clj [opts]
  (b/compile-clj {:basis (b/create-basis {:project "deps.edn"})
                   :class-dir "target/classes"
                   :src-dirs ["src"]
                   :compile-opts {:direct-linking true}}))
;; direct-linking = กำจัด var indirection สำหรับ performance

;; AOT ช่วย:
;; 1. Faster startup time (ไม่ต้อง compile ตอน start)
;; 2. Direct linking performance
;; 3. Java interop ดีขึ้น

;; ข้อเสีย:
;; - Development ลำบากขึ้น
;; - Rebuild ต้องใช้เวลา
;; - ใช้ใน production build เท่านั้น
```

---

## ขั้นตอนที่ 429: Parallelism Patterns

```clojure
;; ===== When to use parallelism =====

;; pmap: parallel map (CPU-bound, N cores)
(defn compress-images [images]
  (doall (pmap compress-image images)))

;; core.async pipeline: pipeline สำหรับ controlled parallelism
(require '[clojure.core.async :as async])

(defn parallel-process [items n process-fn]
  (let [input  (async/chan (count items))
        output (async/chan (count items))]
    (async/pipeline-blocking n output (map process-fn) input)
    (async/onto-chan! input items)
    (async/<!! (async/into [] output))))

;; Fork/join ด้วย reducers
(require '[clojure.core.reducers :as r])

(defn parallel-sum-of-squares [coll]
  (->> coll
       (r/filter even?)
       (r/map #(* % %))
       (r/fold +)))  ; parallelizes across available cores!

;; future + deref สำหรับ scatter-gather
(defn scatter-gather [tasks]
  (let [futures (map #(future (%)) tasks)]
    (map deref futures)))  ; wait for all

;; Promise สำหรับ first-wins race
(defn fastest-result [tasks]
  (let [p (promise)]
    (doseq [task tasks]
      (future
        (let [result (task)]
          (deliver p result))))  ; first delivery wins
    @p))
```

---

## ขั้นตอนที่ 430: Caching Strategies

```clojure
;; ===== Multi-level Caching =====

;; L1: In-process (atom/memoize)
(def l1-cache (atom {}))

;; L2: Shared in-process (Guava/Caffeine)
;; L3: External (Redis)

;; Smart caching with TTL
(defprotocol Cache
  (get-cached [this key])
  (set-cached! [this key value ttl-ms])
  (invalidate! [this key]))

;; Caffeine (Java) in Clojure
(import '[com.github.benmanes.caffeine.cache Caffeine])

(defn create-caffeine-cache [max-size expire-seconds]
  (-> (Caffeine/newBuilder)
      (.maximumSize max-size)
      (.expireAfterWrite expire-seconds java.util.concurrent.TimeUnit/SECONDS)
      (.build)))

(defn cache-get [^com.github.benmanes.caffeine.cache.Cache cache key]
  (.getIfPresent cache key))

(defn cache-put! [^com.github.benmanes.caffeine.cache.Cache cache key value]
  (.put cache key value))

;; Cache-aside pattern
(defn get-user-with-cache [cache db user-id]
  (or (cache-get cache user-id)
      (when-let [user (db/find-user db user-id)]
        (cache-put! cache user-id user)
        user)))
```

---

## สรุป Performance Tips

```
Quick Wins:
===========
1. Type hints ลด reflection
   (set! *warn-on-reflection* true) เพื่อหา reflection
   
2. Transients สำหรับ bulk operations
   (transient {}) → (assoc! ...) → (persistent!)
   
3. Primitives ใน hot loops
   ^long, ^double, unchecked-add, unchecked-multiply
   
4. Transducers แทน map/filter chains
   (into [] (comp (filter f) (map g)) coll)
   
5. reducers สำหรับ parallelism
   (r/fold + (r/map f coll))

JVM Settings:
=============
-Xmx2g -XX:+UseG1GC -server

Measure Before Optimizing:
===========================
(crit/quick-bench ...)
VisualVM / async-profiler
```

---

*Part 15 จาก 100+ | ขั้นตอน 421-450 จาก 1000+*
