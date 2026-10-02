# Part 81: Concurrency Patterns ขั้นสูง
## ขั้นตอนที่ 2401-2430: STM, Agents, Refs, Futures, Thread Pools

---

## บทนำ

Clojure's concurrency model:
- **STM (Software Transactional Memory)** - coordinate mutable state
- **Agents** - async state changes
- **Refs** - coordinated, synchronous state
- **Thread pools** - manage thread lifecycle
- **Promises** - one-time delivery of values

---

## ขั้นตอนที่ 2401: STM ด้วย Refs

```clojure
(ns myapp.concurrency.stm)

;; STM: multiple refs updated atomically
;; Good for: bank transfers, inventory management, game state

;; Bank account example
(def accounts
  {:alice (ref 1000)
   :bob   (ref 500)})

(defn transfer! [from to amount]
  (dosync
    (when (< @(get accounts from) amount)
      (throw (ex-info "Insufficient funds"
                       {:account from :balance @(get accounts from)})))
    (alter (get accounts from) - amount)
    (alter (get accounts to)   + amount)))

;; Safe: retries automatically if conflict
(future (transfer! :alice :bob 200))
(future (transfer! :bob :alice 100))
;; No race conditions!

;; Shopping cart with STM
(def inventory (ref {:widget 100 :gadget 50}))
(def cart      (ref {}))

(defn add-to-cart! [item qty]
  (dosync
    (let [available (get @inventory item 0)]
      (when (< available qty)
        (throw (ex-info "Out of stock" {:item item})))
      (alter inventory update item - qty)
      (alter cart update item (fnil + 0) qty))))

(defn checkout! []
  (dosync
    (let [order @cart]
      (ref-set cart {})
      order)))

;; commute: order-independent updates (faster than alter)
(def visitor-count (ref 0))

(defn record-visit! []
  (dosync
    (commute visitor-count inc)))  ; Can run in any order
```

---

## ขั้นตอนที่ 2402: Agents สำหรับ Async State

```clojure
;; Agents: async, independent state changes
;; Good for: logging, caching, background processing

;; Stats collector agent
(def stats-agent
  (agent {:requests 0
           :errors   0
           :total-latency 0
           :by-path  {}}))

(defn record-request! [path status latency-ms]
  (send stats-agent
    (fn [stats]
      (-> stats
          (update :requests inc)
          (update-in [:by-path path :count] (fnil inc 0))
          (update-in [:by-path path :total-latency] (fnil + 0) latency-ms)
          (cond-> (>= status 500) (update :errors inc))
          (update :total-latency + latency-ms)))))

;; Error handler for agent
(set-error-handler! stats-agent
  (fn [agent error]
    (println "Agent error:" (.getMessage error))))

;; Send-off for blocking operations
(def email-queue (agent []))

(defn queue-email! [email]
  (send-off email-queue
    (fn [queue]
      (send-email! email)  ; This blocks - OK for send-off
      (conj queue {:email email :sent-at (java.time.Instant/now)}))))

;; Wait for agent
(await-for 5000 stats-agent email-queue)

;; Read agent state
@stats-agent
(agent-error stats-agent)  ; nil if no error
```

---

## ขั้นตอนที่ 2403: Thread Pool Management

```clojure
(ns myapp.concurrency.pools)

;; Custom thread pools for different workloads
(defn make-io-pool [n-threads name]
  (java.util.concurrent.Executors/newFixedThreadPool
    n-threads
    (reify java.util.concurrent.ThreadFactory
      (newThread [_ r]
        (doto (Thread. r (str name "-" (System/nanoTime)))
          (.setDaemon true))))))

(defn make-cpu-pool []
  (java.util.concurrent.ForkJoinPool/commonPool))

;; Bounded work queue to prevent memory exhaustion
(defn make-bounded-pool [n-threads queue-size name]
  (java.util.concurrent.ThreadPoolExecutor.
    n-threads n-threads
    60 java.util.concurrent.TimeUnit/SECONDS
    (java.util.concurrent.ArrayBlockingQueue. queue-size)
    (reify java.util.concurrent.ThreadFactory
      (newThread [_ r]
        (doto (Thread. r name) (.setDaemon true))))
    ;; Rejection policy: run in calling thread if queue full
    (java.util.concurrent.ThreadPoolExecutor$CallerRunsPolicy.)))

;; Submit work to pool
(defn submit! [^java.util.concurrent.ExecutorService pool f]
  (.submit pool ^Callable (fn [] (f))))

;; Wait for all futures
(defn wait-all [futures]
  (mapv deref futures))

;; Parallel map with custom pool
(defn pmap-pool [pool f coll]
  (let [futures (mapv #(submit! pool (fn [] (f %))) coll)]
    (mapv deref futures)))

;; Usage: I/O bound tasks with large pool
(def io-pool (make-io-pool 50 "io-worker"))

(defn fetch-all-users [user-ids]
  (pmap-pool io-pool
    #(http-get (str "/users/" %))
    user-ids))
```

---

## ขั้นตอนที่ 2404: Promises and Futures

```clojure
;; Promises: one-time delivery
(defn async-operation []
  (let [result-promise (promise)]
    ;; Start async work
    (future
      (Thread/sleep 1000)
      (deliver result-promise {:data "computed"}))
    
    ;; Return promise immediately
    result-promise))

(def p (async-operation))
;; ... do other work ...
@p  ; Wait and get result

;; Timeout on promise
(defn deref-with-timeout [p timeout-ms default]
  (deref p timeout-ms default))

;; Chain promises
(defn chain-async [f & args]
  (let [p (promise)]
    (future
      (try
        (deliver p {:ok (apply f args)})
        (catch Exception e
          (deliver p {:error e}))))
    p))

;; Non-blocking with callbacks
(defn async-handler [request callback]
  (future
    (let [result (process-request request)]
      (callback result))))

;; Completion stage (Java 8+)
(defn completable-future [f]
  (java.util.concurrent.CompletableFuture/supplyAsync
    (reify java.util.function.Supplier
      (get [_] (f)))))

(defn then-apply [^java.util.concurrent.CompletableFuture cf f]
  (.thenApply cf (reify java.util.function.Function
                    (apply [_ v] (f v)))))
```

---

## ขั้นตอนที่ 2405: Throttling and Debouncing

```clojure
;; Rate-limiting execution
(defn make-throttle [max-per-second]
  (let [tokens     (atom max-per-second)
        last-refill (atom (System/currentTimeMillis))]
    (fn [f]
      (loop []
        (let [now    (System/currentTimeMillis)
               elapsed (- now @last-refill)]
          ;; Refill tokens
          (when (>= elapsed 1000)
            (reset! tokens max-per-second)
            (reset! last-refill now))
          
          ;; Try to consume token
          (if (> @tokens 0)
            (do
              (swap! tokens dec)
              (f))
            (do
              (Thread/sleep 10)
              (recur))))))))

;; Debounce: coalesce rapid calls
(defn make-debounce [delay-ms f]
  (let [timer (atom nil)]
    (fn [& args]
      (when @timer
        (future-cancel @timer))
      (reset! timer
        (future
          (Thread/sleep delay-ms)
          (apply f args))))))

;; Memoize with TTL
(defn memoize-ttl [f ttl-ms]
  (let [cache (atom {})]
    (fn [& args]
      (let [now     (System/currentTimeMillis)
            cached  (get @cache args)
            expired? (or (nil? cached)
                          (> (- now (:ts cached)) ttl-ms))]
        (if expired?
          (let [result (apply f args)]
            (swap! cache assoc args {:value result :ts now})
            result)
          (:value cached))))))
```

---

*Part 81 จาก 100+ | ขั้นตอน 2401-2430 จาก 1000+*
