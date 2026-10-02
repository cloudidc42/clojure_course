# Part 83: Java Interop ขั้นสูง
## ขั้นตอนที่ 2461-2490: Java Libraries, Reflection, Type Hints, Native Methods

---

## บทนำ

Clojure บน JVM สามารถใช้ Java libraries ได้ทั้งหมด:
- **Java interop** - call Java methods directly
- **Type hints** - eliminate reflection overhead
- **Implementing interfaces** - reify/proxy
- **Macros for Java** - simplify verbose Java APIs
- **Native libraries** - JNI/Panama

---

## ขั้นตอนที่ 2461: Java Interop Basics

```clojure
(ns myapp.jvm)

;; Static methods
(Math/sqrt 16.0)
(System/currentTimeMillis)
(java.util.UUID/randomUUID)

;; Instance creation and methods
(def sb (StringBuilder.))
(.append sb "Hello")
(.append sb " World")
(.toString sb)  ; => "Hello World"

;; Field access
(.-MAX_VALUE Integer)
(.-length "hello")  ; => 5

;; Chaining methods
(.. "Hello World"
    toLowerCase
    (replace "world" "clojure")
    trim)

;; doto: call multiple methods on same object
(doto (java.util.HashMap.)
  (.put "key1" "value1")
  (.put "key2" "value2")
  (.put "key3" "value3"))

;; Import classes
(import [java.util Date Calendar]
        [java.io File FileInputStream])

;; Type checking
(instance? String "hello")  ; => true
(class "hello")              ; => java.lang.String
```

---

## ขั้นตอนที่ 2462: Type Hints สำหรับ Performance

```clojure
;; Type hints eliminate reflection (5-100x faster)
;; Use (:require [clojure.core :refer [type]])

;; Without type hint: reflection at runtime
(defn slow-length [s]
  (.length s))  ; Reflection: JVM must check type at runtime

;; With type hint: direct method call
(defn fast-length [^String s]
  (.length s))  ; No reflection

;; Return type hint
(defn ^StringBuilder append-all [^StringBuilder sb items]
  (doseq [item items]
    (.append sb (str item)))
  sb)

;; Primitive type hints
(defn fast-sum [^longs arr]
  (areduce arr i acc 0
           (+ acc (aget arr i))))

;; Benchmark: check for reflection warnings
(set! *warn-on-reflection* true)  ; In REPL

;; Common type hints
(defn process-file [^java.io.File f]
  {:name  (.getName f)
   :size  (.length f)
   :dir?  (.isDirectory f)})

;; Array operations with type hints
(defn array-sum [^"[D" arr]  ; double array
  (let [n (alength arr)]
    (loop [i 0 sum 0.0]
      (if (< i n)
        (recur (inc i) (+ sum (aget arr i)))
        sum))))
```

---

## ขั้นตอนที่ 2463: Working with Java Collections

```clojure
;; Convert between Clojure and Java collections
(vec (java.util.Arrays/asList 1 2 3))        ; Java -> Clojure
(into-array Integer [1 2 3])                   ; Clojure -> Java array
(java.util.ArrayList. [1 2 3])                 ; Clojure -> ArrayList

;; Java Streams API (Java 8+)
(defn java-stream-example [coll]
  (-> (java.util.Arrays/stream (into-array Integer coll))
      (.filter (reify java.util.function.IntPredicate
                  (test [_ i] (even? i))))
      (.map (reify java.util.function.IntUnaryOperator
               (applyAsInt [_ i] (* i i))))
      (.sum)))

;; Convert Java Optional
(defn optional->maybe [^java.util.Optional opt]
  (when (.isPresent opt) (.get opt)))

;; Java Map operations
(defn java-map->clj [^java.util.Map m]
  (into {} (for [[k v] m] [(keyword k) v])))

;; Persistent collections implement Java interfaces
;; so they work with Java APIs expecting java.util.List etc.
(java.util.Collections/sort (ArrayList. [3 1 2]))

;; Interop with Clojure's java.io wrappers
(slurp "file.txt")            ; Reads file
(spit "file.txt" "content")   ; Writes file
(clojure.java.io/copy src dst) ; Stream copy
```

---

## ขั้นตอนที่ 2464: Implementing Java Interfaces

```clojure
;; Implement Java interfaces in Clojure

;; Runnable (for threads)
(def my-task
  (reify Runnable
    (run [_] (println "Running in thread"))))

(doto (Thread. my-task)
  (.start))

;; Comparator
(def by-name
  (reify java.util.Comparator
    (compare [_ a b]
      (compare (:name a) (:name b)))))

;; AutoCloseable (for try-with-resources equivalent)
(defn make-closeable-resource [name]
  (reify java.lang.AutoCloseable
    (close [_] (println "Closing" name))))

;; Use with with-open
(with-open [r (make-closeable-resource "my-resource")]
  (println "Using resource"))
;; => "Closing my-resource" automatically called

;; Iterator
(defn clj-seq->iterator [coll]
  (let [state (atom coll)]
    (reify java.util.Iterator
      (hasNext [_] (boolean (seq @state)))
      (next    [_]
        (let [item (first @state)]
          (swap! state rest)
          item)))))

;; Callable (for ExecutorService)
(defn make-callable [f]
  (reify java.util.concurrent.Callable
    (call [_] (f))))
```

---

## ขั้นตอนที่ 2465: JVM Tuning for Clojure

```clojure
;; JVM configuration for production Clojure apps

;; Startup script
;; java \
;;   -server \
;;   -Xmx2g \
;;   -Xms512m \
;;   -XX:+UseG1GC \
;;   -XX:MaxGCPauseMillis=200 \
;;   -XX:+UseContainerSupport \
;;   -XX:MaxRAMPercentage=75.0 \
;;   -XX:+HeapDumpOnOutOfMemoryError \
;;   -XX:HeapDumpPath=/tmp/heap-dump.hprof \
;;   -Dclojure.server.repl="{:port 7888}" \
;;   -jar myapp.jar

;; Check JVM info at runtime
(defn jvm-info []
  (let [runtime (Runtime/getRuntime)
        mem     (java.lang.management.ManagementFactory/getMemoryMXBean)]
    {:processors  (.availableProcessors runtime)
     :heap-used   (.getUsed (.getHeapMemoryUsage mem))
     :heap-max    (.getMax (.getHeapMemoryUsage mem))
     :gc-stats    (for [gc (java.lang.management.ManagementFactory/getGarbageCollectorMXBeans)]
                     {:name           (.getName gc)
                      :collection-count (.getCollectionCount gc)
                      :collection-time  (.getCollectionTime gc)})}))

;; Force GC (use sparingly!)
(defn suggest-gc! []
  (System/gc))

;; Thread dump for debugging
(defn thread-dump []
  (let [threads (java.lang.management.ManagementFactory/getThreadMXBean)]
    (map (fn [info]
            {:id       (.getThreadId info)
             :name     (.getThreadName info)
             :state    (str (.getThreadState info))
             :blocked? (.isBlocked info)})
         (.dumpAllThreads threads false false))))
```

---

*Part 83 จาก 100+ | ขั้นตอน 2461-2490 จาก 1000+*
