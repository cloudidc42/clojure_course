# Part 48: Concurrency และ Parallelism
## ขั้นตอนที่ 1411-1440: STM, Agents, Fork/Join, Reducers, Parallel Processing

---

## บทนำ

Concurrency ใน Clojure:
- **STM (Software Transactional Memory)** - refs, dosync
- **Agents** - async state updates
- **Fork/Join** - parallel computation
- **Reducers** - parallel reduce
- **Locking patterns** - when STM isn't enough

---

## ขั้นตอนที่ 1411: STM ด้วย Refs

```clojure
(ns myapp.stm)

;; Refs: coordinated synchronous state change
(def bank-balance-a (ref 1000.0))
(def bank-balance-b (ref 500.0))

;; Transfer money atomically (STM)
(defn transfer! [from to amount]
  (dosync
    (when (< @from amount)
      (throw (ex-info "Insufficient funds"
                       {:balance @from :amount amount})))
    (alter from - amount)
    (alter to   + amount)))

;; Concurrent transfers won't corrupt state
(transfer! bank-balance-a bank-balance-b 200.0)
;; a = 800, b = 700

;; STM retries automatically on conflict
;; No explicit locking needed!

;; ref-set: replace entire value
(dosync (ref-set bank-balance-a 0.0))

;; commute: order-independent update (faster)
(def visitor-count (ref 0))
(defn record-visit! []
  (dosync (commute visitor-count inc)))

;; Multiple refs in one transaction
(def inventory (ref {:apples 10 :bananas 5}))
(def cart      (ref {:apples 0  :bananas 0}))
(def revenue   (ref 0.0))

(defn purchase! [item qty price]
  (dosync
    (let [available (get @inventory item 0)]
      (when (< available qty)
        (throw (ex-info "Out of stock" {:item item :qty qty})))
      (alter inventory update item - qty)
      (alter cart      update item + qty)
      (alter revenue   + (* qty price)))))
```

---

## ขั้นตอนที่ 1412: Agents สำหรับ Async Updates

```clojure
(ns myapp.agents)

;; Agent: asynchronous independent state update
(def event-log (agent []))
(def metrics   (agent {:requests 0 :errors 0 :total-ms 0}))

;; Send action to agent (async, returns immediately)
(defn log-event! [event]
  (send event-log conj
    (assoc event :timestamp (java.time.Instant/now))))

;; Send-off: for blocking operations (uses unbounded thread pool)
(defn update-metrics! [request-ms error?]
  (send-off metrics
    (fn [current]
      (-> current
          (update :requests inc)
          (update :errors   (if error? inc identity))
          (update :total-ms + request-ms)))))

;; Wait for all agents to finish
(await event-log metrics)

;; Error handling for agents
(defn handle-agent-error! [agent err]
  (println "Agent error:" (.getMessage err))
  (restart-agent agent @agent :clear-actions true))

(set-error-handler! event-log handle-agent-error!)
(set-error-mode! event-log :continue)  ; don't stop on errors

;; Get agent state
(deref event-log)   ; or @event-log

;; Agent-based work queue
(def work-queue (agent []))

(defn submit-work! [task]
  (send-off work-queue
    (fn [queue]
      (process-task! task)
      (rest queue))))

;; Pipeline with agents
(def pipeline-state (agent {:stage :input :processed 0 :errors []}))

(defn advance-pipeline! [input]
  (send pipeline-state
    (fn [state]
      (try
        (let [result (process (:stage state) input)]
          (-> state
              (update :processed inc)
              (assoc :last-result result)))
        (catch Exception e
          (update state :errors conj {:input input :error (.getMessage e)}))))))
```

---

## ขั้นตอนที่ 1413: Fork/Join สำหรับ CPU-bound Work

```clojure
(ns myapp.parallel
  (:import [java.util.concurrent ForkJoinPool RecursiveTask]))

;; Custom RecursiveTask
(defn parallel-sum [^longs arr threshold]
  (let [n (alength arr)]
    (if (<= n threshold)
      ;; Base case: compute directly
      (loop [i 0 total 0]
        (if (= i n)
          total
          (recur (inc i) (+ total (aget arr i)))))
      
      ;; Recursive case: fork and join
      (let [mid   (quot n 2)
            left  (Arrays/copyOfRange arr 0 mid)
            right (Arrays/copyOfRange arr mid n)
            
            left-task  (future (parallel-sum left threshold))
            right-task (future (parallel-sum right threshold))]
        (+ @left-task @right-task)))))

;; Parallel map using ForkJoinPool
(defn pmap-chunked [f coll chunk-size]
  (let [pool  (ForkJoinPool/commonPool)
        chunks (partition-all chunk-size coll)]
    (->> chunks
         (map (fn [chunk]
                (.submit pool ^java.util.concurrent.Callable
                  (fn [] (mapv f chunk)))))
         (mapcat #(.get %)))))

;; Parallel sort
(defn parallel-sort [coll comparator]
  (let [arr (into-array Object coll)]
    (java.util.Arrays/parallelSort arr comparator)
    (vec arr)))

;; Parallel prefix sum (scan)
(defn parallel-prefix-sum [nums]
  (let [arr    (int-array nums)
        result (int-array (count nums))]
    (java.util.Arrays/parallelPrefix arr (fn [a b] (+ a b)))
    (vec arr)))
```

---

## ขั้นตอนที่ 1414: Reducers สำหรับ Parallel Collections

```clojure
(ns myapp.reducers
  (:require [clojure.core.reducers :as r]))

;; Reducers: parallel fold over collections

;; Sequential (slow for large collections)
(defn sum-sequential [coll]
  (reduce + 0 coll))

;; Parallel with reducers
(defn sum-parallel [coll]
  (r/fold + coll))

;; Parallel map+filter+reduce
(defn process-orders [orders]
  (r/fold
    +         ; combine results
    (r/map    :total
              (r/filter #(= :completed (:status %)) orders))))

;; fold with custom combiner
(defn parallel-stats [numbers]
  (r/fold
    ;; Combine two partial results
    (fn [a b]
      {:count (+ (:count a) (:count b))
       :sum   (+ (:sum a)   (:sum b))
       :min   (min (:min a) (:min b))
       :max   (max (:max a) (:max b))})
    
    ;; Reduce single element
    (fn [acc x]
      {:count (inc (:count acc))
       :sum   (+ (:sum acc) x)
       :min   (min (:min acc) x)
       :max   (max (:max acc) x)})
    
    ;; Initial value
    {:count 0 :sum 0 :min Long/MAX_VALUE :max Long/MIN_VALUE}
    
    numbers))

;; Result includes avg
(defn with-avg [stats]
  (assoc stats :avg (/ (:sum stats) (double (:count stats)))))
```

---

## ขั้นตอนที่ 1415: Locking Patterns

```clojure
(ns myapp.locking)

;; Java locks when STM isn't enough (external resources, I/O)
(import '[java.util.concurrent.locks ReentrantReadWriteLock])

(def rw-lock (ReentrantReadWriteLock.))

(defn with-read-lock [f]
  (let [lock (.readLock rw-lock)]
    (.lock lock)
    (try
      (f)
      (finally
        (.unlock lock)))))

(defn with-write-lock [f]
  (let [lock (.writeLock rw-lock)]
    (.lock lock)
    (try
      (f)
      (finally
        (.unlock lock)))))

;; Read/write cache
(def cache (atom {}))
(def cache-lock (ReentrantReadWriteLock.))

(defn cache-get [key]
  (with-read-lock #(get @cache key)))

(defn cache-put! [key value]
  (with-write-lock #(swap! cache assoc key value)))

;; Semaphore for rate limiting
(import '[java.util.concurrent Semaphore])

(def concurrency-limit (Semaphore. 10))  ; max 10 concurrent

(defn with-concurrency-limit [f]
  (.acquire concurrency-limit)
  (try
    (f)
    (finally
      (.release concurrency-limit))))

;; Example: limit concurrent DB connections
(defn query-with-limit [query]
  (with-concurrency-limit
    #(db/execute! query)))
```

---

## ขั้นตอนที่ 1416: Async Patterns ด้วย Promises and Futures

```clojure
(ns myapp.async-patterns)

;; Promise: one-time delivery
(defn async-fetch [url]
  (let [p (promise)]
    (future
      (try
        (deliver p {:ok (slurp url)})
        (catch Exception e
          (deliver p {:error (.getMessage e)}))))
    p))

;; Timeout pattern
(defn with-timeout [timeout-ms p default]
  (let [result (deref p timeout-ms ::timeout)]
    (if (= result ::timeout) default result)))

;; Parallel requests with timeout
(defn fetch-all-with-timeout [urls timeout-ms]
  (let [promises (map async-fetch urls)]
    (map #(with-timeout timeout-ms % {:error "timeout"}) promises)))

;; Future composition
(defn process-async [item]
  (future
    (-> item
        fetch-data
        transform-data
        save-result!)))

;; Wait for multiple futures
(defn wait-all [futures]
  (mapv deref futures))

;; First completed wins
(defn race [& fns]
  (let [result (promise)]
    (doseq [f fns]
      (future
        (try
          (when-not (realized? result)
            (deliver result (f)))
          (catch Exception _))))
    (deref result 5000 nil)))
```

---

## Project: Concurrent Order Processing

```clojure
(ns order.concurrent
  (:require [clojure.core.async :as async]))

;; Concurrent order processor with backpressure
(defn create-order-processor [db concurrency]
  (let [input-ch    (async/chan 1000)
        result-ch   (async/chan 1000)
        
        ;; Worker pool
        workers     (repeatedly concurrency
                      (fn []
                        (async/go-loop []
                          (when-let [order (async/<! input-ch)]
                            (let [result
                                  (try
                                    {:ok (process-order! db order)}
                                    (catch Exception e
                                      {:error (.getMessage e) :order order}))]
                              (async/>! result-ch result)
                              (recur))))))]
    
    {:input  input-ch
     :output result-ch
     :stop!  (fn []
               (async/close! input-ch)
               (doseq [w workers] (async/<!! w)))}))

;; STM-based inventory management
(def inventory (ref {}))

(defn reserve-items! [items]
  (dosync
    (doseq [{:keys [product-id qty]} items]
      (let [available (get @inventory product-id 0)]
        (when (< available qty)
          (throw (ex-info "Insufficient inventory"
                           {:product-id product-id
                            :available  available
                            :requested  qty})))
        (alter inventory update product-id - qty)))))

(defn release-items! [items]
  (dosync
    (doseq [{:keys [product-id qty]} items]
      (alter inventory update product-id (fnil + 0) qty))))
```

---

*Part 48 จาก 100+ | ขั้นตอน 1411-1440 จาก 1000+*
