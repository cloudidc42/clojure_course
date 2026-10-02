# Part 41: Stream Processing
## ขั้นตอนที่ 1201-1230: core.async Pipelines, Manifold, Kafka Streams, Real-time Analytics

---

## บทนำ

Stream processing ด้วย Clojure:
- **core.async** - pipelines, transducers, backpressure
- **Manifold** - deferred values, streams
- **Kafka Streams** - stateful stream processing
- **Real-time Analytics** - aggregations, windowing

---

## ขั้นตอนที่ 1201: core.async Pipelines

```clojure
(ns myapp.pipeline
  (:require [clojure.core.async :as async]))

;; Basic pipeline with transducers
(defn create-pipeline [input-ch]
  (let [;; Stage 1: Parse
        parsed-ch  (async/chan 100)
        ;; Stage 2: Validate
        valid-ch   (async/chan 100)
        ;; Stage 3: Enrich
        enriched-ch (async/chan 100)
        ;; Stage 4: Output
        output-ch  (async/chan 100)]
    
    ;; Parse stage
    (async/pipeline
      4  ; parallelism
      parsed-ch
      (map (fn [raw]
             (try
               {:status :ok :data (parse-json raw)}
               (catch Exception e
                 {:status :error :msg (.getMessage e)}))))
      input-ch)
    
    ;; Validate stage
    (async/pipeline
      2
      valid-ch
      (filter #(= :ok (:status %)))
      parsed-ch)
    
    ;; Enrich stage (async IO)
    (async/pipeline-async
      8  ; concurrent
      enriched-ch
      (fn [item result-ch]
        (async/go
          (let [enriched (async/<! (fetch-user-data (:user-id (:data item))))]
            (async/>! result-ch (assoc-in item [:data :user] enriched)))))
      valid-ch)
    
    [output-ch enriched-ch]))

;; Backpressure with sliding buffer
(def high-priority-ch   (async/chan 1000))
(def medium-priority-ch (async/chan (async/sliding-buffer 500)))  ; drops oldest
(def low-priority-ch    (async/chan (async/dropping-buffer 100))) ; drops newest
```

---

## ขั้นตอนที่ 1202: Transducer Pipelines

```clojure
(ns myapp.xf
  (:require [clojure.core.async :as async]))

;; Composable transducers for stream processing
(def event-pipeline-xf
  (comp
    ;; Filter only valid events
    (filter #(contains? % :event-type))
    
    ;; Parse timestamps
    (map #(update % :timestamp java.time.Instant/parse))
    
    ;; Filter recent events (last 5 minutes)
    (filter #(let [age (- (System/currentTimeMillis)
                           (.toEpochMilli (:timestamp %)))]
               (< age (* 5 60 1000))))
    
    ;; Add processing metadata
    (map #(assoc % :processed-at (java.time.Instant/now)
                    :processor-id (System/getenv "HOSTNAME")))
    
    ;; Group by type (partition into batches for efficiency)
    (partition-by :event-type)
    
    ;; Flatten back
    cat))

;; Apply transducer to channel
(defn process-events! [input-ch output-ch]
  (async/pipeline
    (-> (Runtime/getRuntime) .availableProcessors (* 2))
    output-ch
    event-pipeline-xf
    input-ch))

;; Stateful transducers
(defn rate-limit-xf [max-per-second]
  (fn [rf]
    (let [count     (volatile! 0)
          window-start (volatile! (System/currentTimeMillis))]
      (fn
        ([] (rf))
        ([result] (rf result))
        ([result input]
         (let [now    (System/currentTimeMillis)
               window (- now @window-start)]
           (when (> window 1000)
             (vreset! count 0)
             (vreset! window-start now))
           (if (< @count max-per-second)
             (do (vswap! count inc)
                 (rf result input))
             result)))))))

;; Deduplication transducer
(defn deduplicate-xf [key-fn]
  (fn [rf]
    (let [seen (volatile! (java.util.HashSet.))]
      (fn
        ([] (rf))
        ([result] (rf result))
        ([result input]
         (let [k (key-fn input)]
           (if (.contains @seen k)
             result
             (do (.add @seen k)
                 (rf result input)))))))))
```

---

## ขั้นตอนที่ 1203: Windowing Aggregations

```clojure
(ns myapp.windowing
  (:require [clojure.core.async :as async]))

;; Tumbling window (non-overlapping)
(defn tumbling-window [ch window-ms]
  (let [out (async/chan 100)]
    (async/go-loop [window []
                     deadline (+ (System/currentTimeMillis) window-ms)]
      (let [timeout-ch (async/timeout (- deadline (System/currentTimeMillis)))]
        (async/alt!
          ch       ([item]
                    (if item
                      (recur (conj window item) deadline)
                      ;; Channel closed, flush
                      (when (seq window)
                        (async/>! out window))))
          timeout-ch ([_]
                      (when (seq window)
                        (async/>! out window))
                      (recur [] (+ (System/currentTimeMillis) window-ms))))))
    out))

;; Sliding window (overlapping)
(defn sliding-window [ch window-size step]
  (let [out (async/chan 100)]
    (async/go-loop [buffer (java.util.ArrayDeque.)]
      (when-let [item (async/<! ch)]
        (.addLast buffer item)
        (when (>= (.size buffer) window-size)
          (async/>! out (vec (.toArray buffer)))
          ;; Advance by step
          (dotimes [_ step]
            (.pollFirst buffer)))
        (recur buffer)))
    out))

;; Aggregate window results
(defn aggregate-window [events]
  {:count     (count events)
   :sum       (reduce + (map :value events))
   :avg       (/ (reduce + (map :value events)) (count events))
   :min       (apply min (map :value events))
   :max       (apply max (map :value events))
   :window-end (java.time.Instant/now)})

;; Real-time metrics pipeline
(defn metrics-pipeline! [event-ch]
  (let [windowed-ch (tumbling-window event-ch 10000)]  ; 10s windows
    (async/go-loop []
      (when-let [window (async/<! windowed-ch)]
        (let [agg (aggregate-window window)]
          (metrics/record-window! agg)
          (println "Window metrics:" agg))
        (recur)))))
```

---

## ขั้นตอนที่ 1204: Manifold Streams

```clojure
;; deps.edn
;; {:deps {manifold/manifold {:mvn/version "0.4.2"}}}

(ns myapp.manifold
  (:require [manifold.stream :as s]
            [manifold.deferred :as d]))

;; Manifold streams
(def raw-events (s/stream 1000))
(def processed-events (s/stream 1000))

;; Connect with transformation
(s/connect-via
  raw-events
  (fn [event]
    (d/chain
      (d/future (parse-event event))
      (fn [parsed]
        (s/put! processed-events parsed))))
  processed-events)

;; Async processing with deferred
(defn process-async [item]
  (d/chain
    ;; Step 1: validate (immediate)
    (d/future (validate item))
    
    ;; Step 2: enrich from DB (async)
    (fn [valid]
      (when valid
        (fetch-enrichment-data (:id item))))
    
    ;; Step 3: transform
    (fn [enrichment]
      (assoc item :enriched enrichment :processed-at (java.time.Instant/now)))))

;; Fan-out to multiple consumers
(defn fan-out! [source consumers]
  (let [broadcast (s/stream 100)]
    (s/connect source broadcast {:upstream? true})
    (doseq [consumer consumers]
      (let [consumer-ch (s/stream 100)]
        (s/connect broadcast consumer-ch)
        (s/consume consumer consumer-ch)))))

;; Usage
(fan-out! processed-events
  [analytics/handle-event!
   audit-log/record!
   notification/maybe-notify!])

;; Rate limiting with manifold
(defn rate-limited-stream [source rate-per-second]
  (let [interval (/ 1000 rate-per-second)]
    (s/transform
      (fn [xf]
        (fn
          ([] (xf))
          ([result] (xf result))
          ([result item]
           (Thread/sleep interval)
           (xf result item))))
      source)))
```

---

## ขั้นตอนที่ 1205: Real-time Analytics Engine

```clojure
(ns analytics.engine
  (:require [clojure.core.async :as async]))

;; In-memory time-series store
(def metrics-store
  (atom {:counters  {}   ; "event-type" -> count
          :gauges    {}   ; "metric-name" -> current value
          :histograms {}  ; "metric-name" -> [values]
          :windows   {}})) ; "metric-name" -> time-windowed

;; Record different metric types
(defn inc-counter! [name & [labels]]
  (let [key (str name (when labels (str ":" (pr-str labels))))]
    (swap! metrics-store update-in [:counters key] (fnil inc 0))))

(defn set-gauge! [name value]
  (swap! metrics-store assoc-in [:gauges name]
    {:value value :updated-at (System/currentTimeMillis)}))

(defn record-histogram! [name value]
  (swap! metrics-store update-in [:histograms name]
    (fnil conj []) value))

;; Real-time aggregation pipeline
(defn start-analytics! [event-ch]
  (async/go-loop [events-this-second []
                   window-start (System/currentTimeMillis)]
    (let [timeout-ch (async/timeout 100)]  ; 100ms tick
      (async/alt!
        event-ch ([event]
                  (when event
                    ;; Process event immediately
                    (case (:type event)
                      :page-view   (inc-counter! "page_views" {:page (:page event)})
                      :purchase    (do (inc-counter! "purchases")
                                       (record-histogram! "order_value" (:amount event)))
                      :user-login  (inc-counter! "logins"))
                    (recur (conj events-this-second event) window-start)))
        
        timeout-ch ([_]
                    (let [now     (System/currentTimeMillis)
                          elapsed (- now window-start)]
                      ;; Every second, compute rates
                      (when (> elapsed 1000)
                        (let [rate (/ (count events-this-second)
                                      (/ elapsed 1000.0))]
                          (set-gauge! "events_per_second" rate)))
                      (recur [] now)))))))

;; Query functions
(defn get-counter [name] (get-in @metrics-store [:counters name] 0))
(defn get-gauge   [name] (get-in @metrics-store [:gauges   name]))
(defn get-percentile [name p]
  (let [values (sort (get-in @metrics-store [:histograms name] []))]
    (when (seq values)
      (let [idx (int (* p (count values)))]
        (nth values idx)))))
```

---

## Project: Real-time Event Analytics

```clojure
(ns analytics.realtime
  (:require [clojure.core.async :as async]
            [taoensso.carmine :as car]))

;; Event sources: HTTP, WebSocket, Kafka
(def event-ch (async/chan (async/sliding-buffer 10000)))

;; Multi-stage processing
(defn start-analytics-pipeline! []
  ;; Stage 1: Deduplication
  (let [dedup-ch (async/chan 5000 (deduplicate-xf :event-id))]
    (async/pipe event-ch dedup-ch)
    
    ;; Stage 2: Enrichment
    (let [enriched-ch (async/chan 2000)]
      (async/pipeline-async 16 enriched-ch
        (fn [event result-ch]
          (async/go
            (let [user (async/<! (cache/get-user (:user-id event)))]
              (async/>! result-ch (assoc event :user user)))))
        dedup-ch)
      
      ;; Stage 3: Fan-out to handlers
      (let [mult (async/mult enriched-ch)]
        
        ;; Analytics aggregation
        (let [agg-ch (async/tap mult (async/chan 2000))]
          (async/go-loop []
            (when-let [e (async/<! agg-ch)]
              (analytics/record-event! e)
              (recur))))
        
        ;; Real-time dashboard push
        (let [dash-ch (async/tap mult (async/chan 100))]
          (async/go-loop []
            (when-let [e (async/<! dash-ch)]
              (websocket/broadcast! "analytics" e)
              (recur))))))))

(start-analytics-pipeline!)
```

---

*Part 41 จาก 100+ | ขั้นตอน 1201-1230 จาก 1000+*
