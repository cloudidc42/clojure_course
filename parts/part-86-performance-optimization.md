# Part 86: Performance Optimization ขั้นสูง
## ขั้นตอนที่ 2551-2580: Profiling, Benchmarking, Memory, Lazy Sequences

---

## บทนำ

Performance tuning ใน Clojure:
- **Profiling** - find bottlenecks with VisualVM/async-profiler
- **Benchmarking** - criterium for accurate measurements
- **Transducers** - eliminate intermediate collections
- **Primitive operations** - avoid boxing
- **Memory optimization** - reduce allocation pressure

---

## ขั้นตอนที่ 2551: Benchmarking ด้วย Criterium

```clojure
(ns myapp.perf.bench
  (:require [criterium.core :as c]))

;; Accurate benchmarking
(defn bench-sum-list [n]
  (c/bench (reduce + (range n))))

;; Quick bench (fewer iterations)
(c/quick-bench (map #(* % %) (range 1000)))

;; Compare implementations
(defn sum-reduce [coll]
  (reduce + coll))

(defn sum-loop [coll]
  (loop [xs coll acc 0]
    (if (seq xs)
      (recur (rest xs) (+ acc (first xs)))
      acc)))

(defn sum-transient [^java.util.List coll]
  (let [n (.size coll)]
    (loop [i 0 acc 0]
      (if (< i n)
        (recur (inc i) (+ acc (.get coll i)))
        acc))))

;; Benchmark all three
(println "reduce:")
(c/quick-bench (sum-reduce (range 10000)))

(println "loop:")
(c/quick-bench (sum-loop (range 10000)))

(println "transient:")
(c/quick-bench (sum-transient (java.util.ArrayList. (range 10000))))

;; With explicit type hints
(set! *warn-on-reflection* true)
(set! *unchecked-math* :warn-on-boxed)
```

---

## ขั้นตอนที่ 2552: Transducers สำหรับ Zero Intermediate Collections

```clojure
;; Normal pipeline: creates 3 intermediate collections
(defn process-normal [data]
  (->> data
       (filter even?)
       (map #(* % %))
       (take 100)
       (reduce +)))

;; Transducer pipeline: no intermediate collections
(defn process-transducer [data]
  (transduce
    (comp
      (filter even?)
      (map #(* % %))
      (take 100))
    +
    data))

;; Custom transducer
(defn chunk-by-n [n]
  (fn [rf]
    (let [buffer (volatile! [])]
      (fn
        ([] (rf))
        ([result]
         (let [buf @buffer]
           (if (seq buf)
             (rf (rf result buf))
             (rf result))))
        ([result input]
         (let [buf (vswap! buffer conj input)]
           (if (= (count buf) n)
             (do (vreset! buffer [])
                 (rf result buf))
             result)))))))

;; Use custom transducer
(into [] (chunk-by-n 3) (range 10))
;; => [[0 1 2] [3 4 5] [6 7 8] [9]]

;; Transducers with core.async channels
(require '[clojure.core.async :as async])

(defn process-events-async [input-ch]
  (let [output-ch (async/chan 100
                    (comp (filter :valid?)
                          (map transform-event)
                          (take-while #(not= :stop (:type %)))))]
    (async/pipe input-ch output-ch)
    output-ch))
```

---

## ขั้นตอนที่ 2553: Primitive Arrays สำหรับ High-Performance

```clojure
;; Use primitive arrays to avoid boxing overhead

;; Double array operations
(defn dot-product [^doubles a ^doubles b]
  (let [n (alength a)]
    (loop [i 0 sum 0.0]
      (if (< i n)
        (recur (inc i) (+ sum (* (aget a i) (aget b i))))
        sum))))

;; Matrix multiplication with arrays
(defn matrix-multiply [^"[[D" a ^"[[D" b]
  (let [rows (alength a)
        cols (alength (aget b 0))
        k    (alength b)
        result (make-array Double/TYPE rows cols)]
    (dotimes [i rows]
      (dotimes [j cols]
        (dotimes [p k]
          (aset result i j
                (+ (aget result i j)
                   (* (aget a i p) (aget b p j)))))))
    result))

;; Fast string operations
(defn fast-join [^java.util.List strings ^String sep]
  (let [sb (StringBuilder.)]
    (doseq [i (range (.size strings))]
      (when (pos? i)
        (.append sb sep))
      (.append sb (.get strings i)))
    (.toString sb)))

;; Efficient number parsing
(defn parse-long-fast [^String s]
  (Long/parseLong s))

;; Unchecked arithmetic (skip overflow checking)
(defn unchecked-sum [a b]
  (unchecked-add a b))
```

---

## ขั้นตอนที่ 2554: Memory Optimization

```clojure
;; Reduce memory allocation

;; 1. Use transients for bulk construction
(defn build-map-transient [pairs]
  (persistent!
    (reduce (fn [m [k v]] (assoc! m k v))
            (transient {})
            pairs)))

;; 2. Use lazy sequences to avoid loading everything
(defn process-large-file [filename]
  (with-open [reader (clojure.java.io/reader filename)]
    (->> (line-seq reader)
         (filter #(clojure.string/starts-with? % "ERROR"))
         (map parse-log-line)
         (take 1000)
         vec)))  ; Force evaluation inside with-open!

;; 3. Avoid head retention in lazy seqs
(defn count-lines-lazy [filename]
  (with-open [reader (clojure.java.io/reader filename)]
    (count (line-seq reader))))  ; Doesn't hold all lines in memory

;; 4. Use records instead of maps for hot paths
(defrecord Point [^double x ^double y])

(defn make-point [x y] (->Point x y))
;; Records: faster field access, less memory than maps

;; 5. Object pooling for expensive objects
(defn make-pool [create-fn max-size]
  (let [pool (java.util.concurrent.ArrayBlockingQueue. max-size)]
    {:borrow  (fn []
                (or (.poll pool)
                    (create-fn)))
     :return! (fn [obj]
                (when (< (.size pool) max-size)
                  (.offer pool obj)))}))

;; Pool of expensive regex matchers
(def pattern-pool
  (make-pool #(re-pattern "[0-9]+") 10))
```

---

## ขั้นตอนที่ 2555: Profiling ด้วย async-profiler

```clojure
;; Profile Clojure applications

;; JVM flags for profiling:
;; -XX:+UnlockDiagnosticVMOptions
;; -XX:+DebugNonSafepoints

;; Using async-profiler via clj-async-profiler
(require '[clj-async-profiler.core :as prof])

;; Profile a function
(prof/profile
  (dotimes [_ 100000]
    (my-expensive-function input)))

;; Generate flame graph
(prof/serve-files 8080)

;; Manual profiling
(defn profile-block [name f]
  (let [start (System/nanoTime)
        result (f)
        end   (System/nanoTime)]
    (println name "took" (/ (- end start) 1e6) "ms")
    result))

;; Sampling-based timing (less overhead than nanoTime)
(defn make-histogram []
  (atom {:count 0 :sum 0 :min Long/MAX_VALUE :max 0 :buckets {}}))

(defn record! [hist value-ms]
  (swap! hist
    (fn [{:keys [count sum min max buckets]}]
      {:count   (inc count)
       :sum     (+ sum value-ms)
       :min     (min min value-ms)
       :max     (max max value-ms)
       :buckets (update buckets
                   (* 10 (quot value-ms 10))
                   (fnil inc 0))})))

(defn percentile [hist p]
  (let [{:keys [count buckets]} @hist
        target (* count (/ p 100))
        sorted (sort-by first buckets)]
    (loop [[[bucket freq] & rest] sorted acc 0]
      (let [new-acc (+ acc freq)]
        (if (or (>= new-acc target) (empty? rest))
          bucket
          (recur rest new-acc))))))
```

---

*Part 86 จาก 100+ | ขั้นตอน 2551-2580 จาก 1000+*
