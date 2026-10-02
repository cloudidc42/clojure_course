# Part 52: core.async ขั้นสูง
## ขั้นตอนที่ 1531-1560: CSP, Pipelines, Error Handling, Patterns, Performance

---

## บทนำ

core.async advanced patterns:
- **CSP** - Communicating Sequential Processes
- **go-loop patterns** - state machines
- **Error propagation** - ผ่าน channels
- **Backpressure** - ควบคุม flow
- **Performance** - thread pool tuning

---

## ขั้นตอนที่ 1531: Advanced Channel Patterns

```clojure
(ns myapp.async
  (:require [clojure.core.async :as async :refer
             [chan go go-loop >! <! >!! <!! close!
              timeout alts! alts!! mult tap pub sub
              pipeline pipeline-async buffer dropping-buffer sliding-buffer]]))

;; Pub/Sub pattern
(defn create-event-bus []
  (let [source-ch (chan (buffer 1000))
        publisher  (pub source-ch :event-type)]
    {:publish!    (fn [event] (async/put! source-ch event))
     :subscribe!  (fn [event-type]
                    (let [ch (chan (buffer 100))]
                      (sub publisher event-type ch)
                      ch))
     :unsubscribe (fn [ch event-type]
                    (async/unsub publisher event-type ch))}))

;; Usage
(def bus (create-event-bus))
(def order-ch ((:subscribe! bus) :order-placed))
(def user-ch  ((:subscribe! bus) :user-registered))

(go-loop []
  (when-let [order (<! order-ch)]
    (println "New order:" (:id order))
    (recur)))

((:publish! bus) {:event-type :order-placed :id "123" :total 500})

;; Merge multiple channels
(defn merge-channels [& channels]
  (let [out (chan 100)]
    (doseq [ch channels]
      (go-loop []
        (when-let [v (<! ch)]
          (>! out v)
          (recur))))
    out))

;; Split channel by predicate
(defn split-channel [pred ch]
  (let [true-ch  (chan 100)
        false-ch (chan 100)]
    (go-loop []
      (when-let [v (<! ch)]
        (if (pred v)
          (>! true-ch v)
          (>! false-ch v))
        (recur)))
    [true-ch false-ch]))
```

---

## ขั้นตอนที่ 1532: State Machine ด้วย go-loop

```clojure
;; Order state machine as a go-loop
(defn order-state-machine [order-id command-ch event-ch]
  (go-loop [state :pending
             order (db/get-order order-id)]
    
    (when-let [command (<! command-ch)]
      (let [[new-state new-order]
            (case [state (:type command)]
              [:pending :confirm]
              (let [updated (update-order! order :status :confirmed)]
                [: confirmed updated])
              
              [:confirmed :ship]
              (let [tracking (:tracking command)
                    updated  (update-order! order
                               :status          :shipped
                               :tracking-number tracking)]
                (>! event-ch {:type :order-shipped :order-id order-id :tracking tracking})
                [:shipped updated])
              
              [:shipped :deliver]
              (let [updated (update-order! order :status :delivered)]
                (>! event-ch {:type :order-delivered :order-id order-id})
                [:delivered updated])
              
              ;; Invalid transition
              (do
                (>! event-ch {:type  :invalid-transition
                               :state state
                               :command (:type command)})
                [state order]))]
        
        (when-not (= new-state :delivered)
          (recur new-state new-order))))))

;; Heartbeat monitor
(defn heartbeat [interval-ms action-fn stop-ch]
  (go-loop []
    (async/alt!
      (async/timeout interval-ms) ([_]
                                    (action-fn)
                                    (recur))
      stop-ch ([_] (println "Heartbeat stopped")))))
```

---

## ขั้นตอนที่ 1533: Error Handling ใน core.async

```clojure
;; Problem: exceptions in go blocks are swallowed!
;; Solution: wrap with error channel

(defn go-safe [ch f]
  "Run f in go block, put {:ok result} or {:error e} to ch"
  (go
    (try
      (>! ch {:ok (f)})
      (catch Exception e
        (>! ch {:error e})))))

;; Safe pipeline
(defn safe-pipeline [in out f]
  (go-loop []
    (when-let [item (<! in)]
      (try
        (let [result (f item)]
          (>! out {:ok result :input item}))
        (catch Exception e
          (>! out {:error (.getMessage e) :input item})))
      (recur))))

;; Error propagation through pipeline
(defn pipeline-with-errors [in f]
  (let [out-ok    (chan 100)
        out-error (chan 100)]
    (go-loop []
      (when-let [item (<! in)]
        (try
          (>! out-ok (f item))
          (catch Exception e
            (>! out-error {:error (.getMessage e) :item item})))
        (recur)))
    [out-ok out-error]))

;; Dead letter queue pattern
(defn with-dead-letter-queue [process-fn dead-letter-ch]
  (fn [item]
    (try
      (process-fn item)
      (catch Exception e
        (async/put! dead-letter-ch
          {:item    item
           :error   (.getMessage e)
           :failed-at (java.time.Instant/now)})))))
```

---

## ขั้นตอนที่ 1534: Backpressure Patterns

```clojure
;; Token bucket rate limiter
(defn token-bucket [rate capacity]
  (let [tokens  (atom capacity)
        last-refill (atom (System/currentTimeMillis))]
    {:acquire
     (fn []
       (loop []
         (let [now     (System/currentTimeMillis)
               elapsed (- now @last-refill)
               new-tok (min capacity
                             (+ @tokens (* rate (/ elapsed 1000.0))))]
           (reset! last-refill now)
           (if (>= new-tok 1)
             (do (swap! tokens - 1) true)
             (do (Thread/sleep 10)
                 (recur))))))
     :tokens tokens}))

;; Rate-limited channel
(defn rate-limited-channel [in-ch rate]
  (let [out-ch  (chan 100)
        bucket  (token-bucket rate 10)]
    (go-loop []
      (when-let [item (<! in-ch)]
        ((:acquire bucket))
        (>! out-ch item)
        (recur)))
    out-ch))

;; Batch accumulator with timeout flush
(defn batch-channel
  "Accumulate items then flush as batch when full or timeout"
  [in-ch batch-size timeout-ms]
  (let [out-ch (chan 100)]
    (go-loop [batch []]
      (let [timeout-ch (timeout timeout-ms)]
        (async/alt!
          in-ch    ([item]
                    (if item
                      (let [new-batch (conj batch item)]
                        (if (>= (count new-batch) batch-size)
                          (do (>! out-ch new-batch)
                              (recur []))
                          (recur new-batch)))
                      ;; closed
                      (when (seq batch)
                        (>! out-ch batch))))
          timeout-ch ([_]
                      (when (seq batch)
                        (>! out-ch batch))
                      (recur [])))))
    out-ch))
```

---

## ขั้นตอนที่ 1535: Thread Pool Management

```clojure
;; core.async thread pools
;; go blocks: fixed thread pool (typically 8 threads)
;; future/thread: cached thread pool

;; Configure go thread pool size
(System/setProperty "clojure.core.async.pool-size" "16")

;; For CPU-bound work: use fork-join pool
(defn cpu-bound-task [items]
  (->> items
       (partition-all (/ (count items) (.availableProcessors (Runtime/getRuntime))))
       (map (fn [chunk]
              (async/thread  ; uses cached pool, not go pool
                (mapv expensive-computation chunk))))
       (mapcat #(async/<!! %))))

;; Metrics for async operations
(defn monitored-channel [name capacity]
  (let [ch      (chan capacity)
        pending (atom 0)]
    (add-watch pending :metrics
      (fn [_ _ _ v]
        (metrics/set-gauge! (str "async." name ".pending") v)))
    {:ch      ch
     :pending pending
     :put!    (fn [v]
                (swap! pending inc)
                (async/put! ch v (fn [_] (swap! pending dec))))
     :take!   (fn [f]
                (async/take! ch
                  (fn [v]
                    (swap! pending dec)
                    (f v))))}))
```

---

## Project: Async Task Queue

```clojure
(ns queue.async
  (:require [clojure.core.async :as async]))

;; Priority task queue
(defn create-task-queue [workers]
  (let [high-priority   (async/chan 100)
        medium-priority (async/chan 100)
        low-priority    (async/chan (async/sliding-buffer 50))
        results-ch      (async/chan 100)]
    
    ;; Workers: prefer high priority
    (dotimes [_ workers]
      (async/go-loop []
        (let [task (async/alt!
                     high-priority   ([t] t)
                     medium-priority ([t] t)
                     low-priority    ([t] t)
                     :priority true)]
          (when task
            (try
              (let [result ((:fn task) (:args task))]
                (async/>! results-ch {:ok result :task-id (:id task)}))
              (catch Exception e
                (async/>! results-ch {:error (.getMessage e) :task-id (:id task)})))
            (recur)))))
    
    {:submit!    (fn [priority task]
                   (case priority
                     :high   (async/put! high-priority task)
                     :medium (async/put! medium-priority task)
                     :low    (async/put! low-priority task)))
     :results    results-ch}))
```

---

*Part 52 จาก 100+ | ขั้นตอน 1531-1560 จาก 1000+*
