# Part 92: Streaming & Real-Time Systems
## ขั้นตอนที่ 2731-2760: Kafka, WebSockets, SSE, Live Dashboards

---

## บทนำ

Real-time data systems:
- **Apache Kafka** - event streaming platform
- **WebSocket** - bidirectional browser communication
- **SSE** - server-sent events for push
- **core.async** - concurrent stream processing
- **Live dashboards** - real-time metrics visualization

---

## ขั้นตอนที่ 2731: Kafka Producer และ Consumer

```clojure
(ns myapp.streaming.kafka
  (:import [org.apache.kafka.clients.producer KafkaProducer ProducerRecord]
           [org.apache.kafka.clients.consumer KafkaConsumer ConsumerConfig]
           [org.apache.kafka.common.serialization StringSerializer StringDeserializer]))

;; Kafka producer
(defn make-producer [bootstrap-servers]
  (KafkaProducer.
    {"bootstrap.servers"  bootstrap-servers
     "key.serializer"     (.getName StringSerializer)
     "value.serializer"   (.getName StringSerializer)
     "acks"               "all"       ; Wait for all replicas
     "retries"            "3"
     "compression.type"   "snappy"
     "linger.ms"          "5"         ; Batch for 5ms
     "batch.size"         "16384"}))

;; Produce event
(defn produce! [^KafkaProducer producer topic key value]
  (let [record (ProducerRecord. topic key (json/encode value))]
    (.send producer record
      (reify org.apache.kafka.clients.producer.Callback
        (onCompletion [_ metadata exception]
          (if exception
            (log/error "Failed to send event" {:error (.getMessage exception)})
            (log/debug "Sent event" {:topic  (.topic metadata)
                                     :offset (.offset metadata)})))))))

;; Kafka consumer
(defn make-consumer [bootstrap-servers group-id topics]
  (let [consumer (KafkaConsumer.
                   {"bootstrap.servers"  bootstrap-servers
                    "group.id"           group-id
                    "key.deserializer"   (.getName StringDeserializer)
                    "value.deserializer" (.getName StringDeserializer)
                    "auto.offset.reset"  "earliest"
                    "enable.auto.commit" "false"})]
    (.subscribe consumer topics)
    consumer))

;; Process messages
(defn consume-loop! [^KafkaConsumer consumer process-fn stop-signal]
  (loop []
    (when-not @stop-signal
      (let [records (.poll consumer (java.time.Duration/ofMillis 1000))]
        (doseq [record records]
          (try
            (process-fn {:topic     (.topic record)
                          :partition (.partition record)
                          :offset    (.offset record)
                          :key       (.key record)
                          :value     (json/parse-string (.value record) true)})
            (catch Exception e
              (log/error "Error processing message" {:error (.getMessage e)}))))
        ;; Manual commit after successful processing
        (.commitSync consumer))
      (recur))))
```

---

## ขั้นตอนที่ 2732: Event Stream Processing

```clojure
(ns myapp.streaming.processor
  (:require [clojure.core.async :as async]))

;; Event processing pipeline with core.async
(defn make-processing-pipeline [kafka-consumer process-fn n-workers]
  (let [input-ch  (async/chan 1000)
        output-ch (async/chan 1000)
        stop!     (atom false)]
    
    ;; Kafka → channel bridge
    (async/thread
      (loop []
        (when-not @stop!
          (let [records (.poll kafka-consumer (java.time.Duration/ofMillis 100))]
            (doseq [record records]
              (async/>!! input-ch {:value (json/parse-string (.value record) true)
                                    :meta  {:topic     (.topic record)
                                            :partition (.partition record)
                                            :offset    (.offset record)}}))
            (.commitSync kafka-consumer)
            (recur)))))
    
    ;; Worker pool: N concurrent processors
    (dotimes [_ n-workers]
      (async/go-loop []
        (when-let [msg (async/<! input-ch)]
          (try
            (let [result (process-fn (:value msg))]
              (async/>! output-ch {:result result :meta (:meta msg)}))
            (catch Exception e
              (log/error "Processing error" {:error (.getMessage e)})))
          (recur))))
    
    {:input-ch  input-ch
     :output-ch output-ch
     :stop!     #(reset! stop! true)}))

;; Windowed aggregation (tumbling window)
(defn tumbling-window [ch window-ms agg-fn]
  (let [out-ch (async/chan 100)]
    (async/go-loop [window [] deadline (+ (System/currentTimeMillis) window-ms)]
      (let [timeout-ch (async/timeout (max 0 (- deadline (System/currentTimeMillis))))
            [val port] (async/alts! [ch timeout-ch])]
        (cond
          ;; Timeout: emit window
          (= port timeout-ch)
          (do
            (when (seq window)
              (async/>! out-ch (agg-fn window)))
            (recur [] (+ (System/currentTimeMillis) window-ms)))
          
          ;; New value
          val
          (recur (conj window val) deadline)
          
          ;; Channel closed
          :else
          (when (seq window)
            (async/>! out-ch (agg-fn window))))))
    out-ch))
```

---

## ขั้นตอนที่ 2733: WebSocket Handler

```clojure
(ns myapp.interface.websocket
  (:require [ring.adapter.jetty9 :as jetty9]
            [clojure.core.async :as async]))

;; Connected clients registry
(defonce connected-clients (atom {}))

(defn ws-handler [req]
  (let [client-id (str (java.util.UUID/randomUUID))
        out-ch    (async/chan 100)]
    
    {:on-open
     (fn [ws]
       (println "Client connected:" client-id)
       (swap! connected-clients assoc client-id {:ws ws :ch out-ch})
       
       ;; Start message pump for this client
       (async/go-loop []
         (when-let [msg (async/<! out-ch)]
           (try
             (jetty9/send! ws (json/encode msg))
             (catch Exception e
               (println "Send failed, disconnecting" client-id)))
           (recur))))
     
     :on-text
     (fn [ws text]
       (let [msg (json/parse-string text true)]
         (handle-client-message! client-id msg)))
     
     :on-close
     (fn [ws status-code reason]
       (println "Client disconnected:" client-id reason)
       (swap! connected-clients dissoc client-id)
       (async/close! out-ch))
     
     :on-error
     (fn [ws e]
       (println "WebSocket error for" client-id (.getMessage e)))}))

;; Broadcast to all connected clients
(defn broadcast! [message]
  (doseq [[_id {:keys [ch]}] @connected-clients]
    (async/put! ch message)))

;; Send to specific client
(defn send-to-client! [client-id message]
  (when-let [{:keys [ch]} (get @connected-clients client-id)]
    (async/put! ch message)))

;; Handle incoming messages
(defmulti handle-client-message!
  (fn [_client-id msg] (:type msg)))

(defmethod handle-client-message! :subscribe
  [client-id {:keys [channel]}]
  (swap! connected-clients update-in [client-id :subscriptions]
         (fnil conj #{}) channel))

(defmethod handle-client-message! :unsubscribe
  [client-id {:keys [channel]}]
  (swap! connected-clients update-in [client-id :subscriptions]
         disj channel))
```

---

## ขั้นตอนที่ 2734: Server-Sent Events (SSE)

```clojure
(ns myapp.interface.sse
  (:require [ring.core.protocols :as rp]))

;; SSE response
(defn sse-stream [ch]
  {:status  200
   :headers {"Content-Type"  "text/event-stream"
              "Cache-Control" "no-cache"
              "Connection"    "keep-alive"
              "X-Accel-Buffering" "no"}  ; nginx: don't buffer
   :body
   (reify rp/StreamableResponseBody
     (write-body-to-stream [_ _response output-stream]
       (with-open [writer (java.io.OutputStreamWriter. output-stream "UTF-8")]
         (let [send-event (fn [data event-type]
                             (when event-type
                               (.write writer (str "event: " event-type "\n")))
                             (.write writer (str "data: " (json/encode data) "\n\n"))
                             (.flush writer))]
           (loop []
             (when-let [event (async/<!! ch)]
               (try
                 (send-event (:data event) (:event event))
                 (recur)
                 (catch java.io.IOException _
                   ;; Client disconnected
                   (async/close! ch))))))))})

;; Live metrics SSE endpoint
(defn metrics-stream-handler [request]
  (let [client-ch (async/chan 100)]
    ;; Subscribe client to metrics updates
    (subscribe-metrics-listener! client-ch)
    
    ;; Send initial state
    (async/put! client-ch {:event "init" :data (current-metrics)})
    
    (sse-stream client-ch)))
```

---

## ขั้นตอนที่ 2735: Real-Time Dashboard

```clojure
;; Live metrics aggregation for dashboard

(defonce dashboard-state
  (atom {:orders-per-minute   0
          :revenue-per-minute  0
          :error-rate          0
          :active-users        0
          :p99-latency         0}))

;; Update metrics from event stream
(defn process-metrics-event! [event]
  (case (:type event)
    :request-completed
    (swap! dashboard-state
      (fn [state]
        (-> state
            (update :requests-per-minute (fnil inc 0))
            (update :total-latency (fnil + 0) (:latency-ms event))
            (cond-> (>= (:status event) 500)
              (update :errors-per-minute (fnil inc 0))))))
    
    :order-placed
    (swap! dashboard-state
      (fn [state]
        (-> state
            (update :orders-per-minute (fnil inc 0))
            (update :revenue-per-minute + (:total event)))))
    
    nil))

;; Periodic metrics reset (sliding window)
(defn start-metrics-window! []
  (async/go-loop []
    (async/<! (async/timeout 60000))  ; 1 minute
    (let [current @dashboard-state]
      (broadcast! {:event "metrics"
                    :data  current})
      (swap! dashboard-state
        #(assoc %
           :orders-per-minute  0
           :revenue-per-minute 0
           :errors-per-minute  0
           :requests-per-minute 0)))
    (recur)))
```

---

*Part 92 จาก 100+ | ขั้นตอน 2731-2760 จาก 1000+*
