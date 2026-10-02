# Part 77: Reactive Programming
## ขั้นตอนที่ 2281-2310: RxClojure, Observable Streams, Backpressure, FRP

---

## บทนำ

Reactive programming patterns:
- **Observable streams** - push-based data flows
- **Backpressure** - control data flow rate
- **Hot vs Cold observables** - shared vs independent streams
- **FRP** - functional reactive programming
- **Reactive DB queries** - live query updates

---

## ขั้นตอนที่ 2281: Event Streams ด้วย core.async

```clojure
(ns myapp.reactive.streams
  (:require [clojure.core.async :as async]))

;; Observable stream abstraction
(defprotocol Observable
  (subscribe! [this subscriber])
  (unsubscribe! [this subscription]))

;; Hot stream: multicasts to all subscribers
(defrecord HotStream [pub-ch topic subscribers]
  Observable
  (subscribe! [this handler]
    (let [sub-ch (async/chan 100)
          sub-id (str (java.util.UUID/randomUUID))]
      (async/sub (:pub pub-ch) topic sub-ch)
      (swap! subscribers assoc sub-id {:ch sub-ch :handler handler})
      (async/go-loop []
        (when-let [event (async/<! sub-ch)]
          (handler event)
          (recur)))
      sub-id))
  
  (unsubscribe! [this sub-id]
    (when-let [{:keys [ch]} (get @subscribers sub-id)]
      (async/close! ch)
      (swap! subscribers dissoc sub-id))))

(defn hot-stream [topic]
  (let [source-ch (async/chan 1000)
        pub       (async/pub source-ch :topic)]
    (->HotStream {:ch source-ch :pub pub} topic (atom {}))))

(defn emit! [stream event]
  (async/put! (:ch (:pub-ch stream))
              (assoc event :topic (:topic stream))))

;; Cold stream: new execution per subscriber
(defrecord ColdStream [producer]
  Observable
  (subscribe! [this handler]
    (let [ch (async/chan 100)]
      ((:producer this) ch)
      (async/go-loop []
        (when-let [value (async/<! ch)]
          (handler value)
          (recur)))
      ch))
  
  (unsubscribe! [_ subscription]
    (async/close! subscription)))

(defn cold-stream [producer-fn]
  (->ColdStream producer-fn))

;; Timer stream
(defn interval-stream [ms]
  (cold-stream
    (fn [ch]
      (async/go-loop [i 0]
        (async/<! (async/timeout ms))
        (if (async/>! ch i)
          (recur (inc i))
          (println "Timer stopped"))))))
```

---

## ขั้นตอนที่ 2282: Stream Operators

```clojure
;; Map operator
(defn stream-map [stream f]
  (cold-stream
    (fn [out-ch]
      (subscribe! stream
        (fn [value]
          (async/put! out-ch (f value)))))))

;; Filter operator
(defn stream-filter [stream pred]
  (cold-stream
    (fn [out-ch]
      (subscribe! stream
        (fn [value]
          (when (pred value)
            (async/put! out-ch value)))))))

;; Merge multiple streams
(defn stream-merge [& streams]
  (cold-stream
    (fn [out-ch]
      (doseq [s streams]
        (subscribe! s
          (fn [value]
            (async/put! out-ch value)))))))

;; Debounce: emit only after quiet period
(defn stream-debounce [stream delay-ms]
  (cold-stream
    (fn [out-ch]
      (let [last-timer (atom nil)]
        (subscribe! stream
          (fn [value]
            (when @last-timer
              (future-cancel @last-timer))
            (reset! last-timer
              (future
                (Thread/sleep delay-ms)
                (async/put! out-ch value)))))))))

;; Buffer: collect into batches
(defn stream-buffer [stream size timeout-ms]
  (cold-stream
    (fn [out-ch]
      (let [buffer (atom [])
            flush! (fn []
                     (when-not (empty? @buffer)
                       (async/put! out-ch @buffer)
                       (reset! buffer [])))]
        (subscribe! stream
          (fn [value]
            (swap! buffer conj value)
            (when (>= (count @buffer) size)
              (flush!))))
        
        ;; Timeout flush
        (async/go-loop []
          (async/<! (async/timeout timeout-ms))
          (flush!)
          (recur))))))

;; Zip: combine latest from two streams
(defn stream-zip [stream1 stream2 f]
  (cold-stream
    (fn [out-ch]
      (let [latest1 (atom nil)
            latest2 (atom nil)]
        (subscribe! stream1
          (fn [v]
            (reset! latest1 v)
            (when @latest2
              (async/put! out-ch (f @latest1 @latest2)))))
        (subscribe! stream2
          (fn [v]
            (reset! latest2 v)
            (when @latest1
              (async/put! out-ch (f @latest1 @latest2)))))))))
```

---

## ขั้นตอนที่ 2283: Backpressure

```clojure
;; Backpressure: producer slows down when consumer is slow

;; Token bucket rate limiter as backpressure signal
(defn make-backpressure-controller [max-in-flight]
  (let [semaphore (java.util.concurrent.Semaphore. max-in-flight)]
    {:acquire! (fn [] (.acquire semaphore))
     :release! (fn [] (.release semaphore))
     :available (fn [] (.availablePermits semaphore))}))

;; Producer with backpressure
(defn throttled-producer [source-seq bp-controller process-fn]
  (async/go-loop [items source-seq]
    (when-let [item (first items)]
      ;; Wait if too many in-flight
      ((:acquire! bp-controller))
      
      ;; Process asynchronously
      (async/go
        (try
          (process-fn item)
          (finally
            ((:release! bp-controller))))
        )
      
      (recur (rest items)))))

;; Bounded channel as backpressure
(defn create-bounded-pipeline [producer-fn consumer-fn buffer-size]
  (let [ch (async/chan buffer-size)]
    ;; Producer
    (async/go
      (producer-fn
        (fn [item]
          ;; Blocks when channel is full
          (async/>!! ch item)))
      (async/close! ch))
    
    ;; Consumer
    (async/go-loop []
      (when-let [item (async/<! ch)]
        (consumer-fn item)
        (recur)))))
```

---

## ขั้นตอนที่ 2284: Reactive UI with Reagent

```clojure
;; Client-side reactive patterns with ClojureScript + Reagent
(ns myapp.ui.reactive
  (:require [reagent.core :as r]
            [clojure.core.async :as async :include-macros true]))

;; Atom-based reactive state
(defonce app-state
  (r/atom {:orders []
            :filters {:status nil :page 1}
            :loading? false}))

;; Computed/derived state
(defn filtered-orders []
  (let [{:keys [orders filters]} @app-state
        status (:status filters)]
    (cond->> orders
      status (filter #(= (name (:status %)) status)))))

;; Event stream from DOM
(defn event-stream [el event-type]
  (let [ch (async/chan 100)]
    (.addEventListener el event-type
      (fn [e] (async/put! ch e)))
    ch))

;; Auto-search with debounce
(defn search-input-component []
  (let [query (r/atom "")]
    (fn []
      [:div
       [:input {:type      "text"
                :value     @query
                :on-change #(reset! query (.. % -target -value))}]
       [search-results-component @query]])))

;; Polling for real-time updates
(defn start-order-polling! [state interval-ms]
  (async/go-loop []
    (async/<! (async/timeout interval-ms))
    (let [orders (async/<! (fetch-orders-async!))]
      (swap! state assoc :orders orders))
    (recur)))
```

---

## Project: Real-time Stock Price Ticker

```clojure
(ns myapp.reactive.stock-ticker)

;; Simulated stock price stream
(defn make-price-stream [symbol initial-price]
  (cold-stream
    (fn [ch]
      (async/go-loop [price initial-price]
        (async/<! (async/timeout 1000))
        (let [change     (* price (+ -0.02 (rand 0.04)))
               new-price  (max 0.01 (+ price change))
               tick       {:symbol    symbol
                            :price     new-price
                            :change    change
                            :change-pct (/ change price)
                            :timestamp (java.time.Instant/now)}]
          (when (async/>! ch tick)
            (recur new-price)))))))

;; Aggregate price streams
(defn portfolio-stream [symbols initial-prices]
  (let [streams (map #(make-price-stream % (get initial-prices %)) symbols)]
    (apply stream-merge streams)))

;; Alert when price changes >5%
(defn price-alert-stream [portfolio-stream threshold]
  (stream-filter portfolio-stream
    #(> (Math/abs (:change-pct %)) threshold)))

;; Usage
(def my-portfolio
  (portfolio-stream ["AAPL" "GOOG" "MSFT"]
                     {"AAPL" 150.0 "GOOG" 2800.0 "MSFT" 300.0}))

(subscribe! my-portfolio
  (fn [tick]
    (println (format "%s: $%.2f (%.2f%%)"
                      (:symbol tick)
                      (:price tick)
                      (* 100 (:change-pct tick))))))

(subscribe! (price-alert-stream my-portfolio 0.05)
  (fn [tick]
    (println "ALERT:" (:symbol tick) "moved" (* 100 (:change-pct tick)) "%")))
```

---

*Part 77 จาก 100+ | ขั้นตอน 2281-2310 จาก 1000+*
