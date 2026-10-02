# Part 62: Message Queues และ Kafka
## ขั้นตอนที่ 1831-1860: Kafka Producer/Consumer, Topics, Partitions, Consumer Groups

---

## บทนำ

Message queues สำหรับ async processing:
- **Kafka** - distributed event streaming
- **RabbitMQ** - traditional message broker
- **Topics/Partitions** - parallel processing
- **Consumer groups** - load balancing
- **Dead letter queues** - error handling

---

## ขั้นตอนที่ 1831: Kafka ด้วย clj-kafka

```clojure
(ns myapp.kafka
  (:require [jackdaw.client :as jc]
            [jackdaw.client.log :as jcl]
            [jackdaw.serdes :as js]))

;; Topic configuration
(def order-events-topic
  {:topic-name         "order-events"
   :partition-count    4
   :replication-factor 3
   :topic-config       {"retention.ms"        "604800000"  ; 7 days
                         "cleanup.policy"      "delete"
                         "compression.type"    "lz4"}})

(def user-events-topic
  {:topic-name         "user-events"
   :partition-count    2
   :replication-factor 3})

;; Producer
(defn create-producer [bootstrap-servers]
  (jc/producer
    {"bootstrap.servers" bootstrap-servers
     "acks"              "all"    ; wait for all replicas
     "retries"           3
     "linger.ms"         5
     "batch.size"        16384
     "compression.type"  "lz4"}
    (js/edn-serde)   ; key serializer
    (js/json-serde))) ; value serializer

;; Send event
(defn send-event! [producer topic key value]
  @(jc/produce! producer
     {:topic-name topic}
     key
     value))

;; Consumer
(defn create-consumer [bootstrap-servers group-id]
  (jc/consumer
    {"bootstrap.servers"  bootstrap-servers
     "group.id"           group-id
     "auto.offset.reset"  "earliest"
     "enable.auto.commit" "false"  ; manual commit
     "max.poll.records"   100}
    (js/edn-serde)
    (js/json-serde)))

;; Process events
(defn start-consumer! [consumer topics handler-fn]
  (jc/subscribe consumer topics)
  (loop []
    (let [records (jc/poll consumer 1000)]
      (doseq [record records]
        (try
          (handler-fn {:key       (:key record)
                        :value     (:value record)
                        :topic     (:topic record)
                        :partition (:partition record)
                        :offset    (:offset record)})
          (catch Exception e
            (println "Error processing:" (.getMessage e)))))
      ;; Commit after successful batch
      (when (seq records)
        (jc/commit-offsets! consumer))
      (recur))))
```

---

## ขั้นตอนที่ 1832: Exactly-Once Processing

```clojure
;; Transactional producer for exactly-once semantics
(defn create-transactional-producer [bootstrap-servers transactional-id]
  (let [producer (jc/producer
                    {"bootstrap.servers"     bootstrap-servers
                     "transactional.id"      transactional-id
                     "enable.idempotence"    "true"
                     "acks"                  "all"}
                    (js/edn-serde)
                    (js/json-serde))]
    (jc/init-transactions! producer)
    producer))

;; Process-transform-produce pattern
(defn process-exactly-once!
  [consumer producer input-topic output-topic handler-fn]
  (loop []
    (let [records (jc/poll consumer 1000)]
      (when (seq records)
        (jc/begin-transaction! producer)
        (try
          (doseq [record records]
            (let [result (handler-fn (:value record))]
              (jc/produce! producer {:topic-name output-topic}
                (:key record) result)))
          
          ;; Commit offsets as part of transaction
          (jc/send-offsets-to-transaction!
            producer consumer "consumer-group-id")
          (jc/commit-transaction! producer)
          
          (catch Exception e
            (jc/abort-transaction! producer)
            (println "Transaction aborted:" (.getMessage e))))))
    (recur)))
```

---

## ขั้นตอนที่ 1833: Consumer Groups

```clojure
;; Multiple consumers in same group = load balanced
;; Each partition assigned to one consumer

;; Consumer group for order processing
(defn start-order-processors! [n-consumers bootstrap-servers]
  (dotimes [i n-consumers]
    (future
      (let [consumer (create-consumer bootstrap-servers "order-processors")]
        (start-consumer! consumer
          [order-events-topic]
          (fn [record]
            (case (:type (:value record))
              :order-placed   (process-new-order! (:value record))
              :order-confirmed (send-confirmation-email! (:value record))
              :order-shipped   (update-tracking! (:value record))
              (println "Unknown event:" (:type (:value record))))))))))

;; Partitioned by user-id for ordered processing per user
(defn publish-user-event! [producer user-id event]
  (send-event! producer "user-events"
    (str user-id)  ; partition key = user-id
    event))        ; all events for same user go to same partition

;; Rebalance listener
(defn make-rebalance-listener []
  (reify org.apache.kafka.clients.consumer.ConsumerRebalanceListener
    (onPartitionsRevoked [_ partitions]
      (println "Revoked partitions:" (map #(.partition %) partitions)))
    (onPartitionsAssigned [_ partitions]
      (println "Assigned partitions:" (map #(.partition %) partitions)))))
```

---

## ขั้นตอนที่ 1834: Dead Letter Queue

```clojure
;; Messages that fail processing go to DLQ
(def dead-letter-topic
  {:topic-name "dead-letter"
   :partition-count 1
   :replication-factor 3})

(defn process-with-dlq! [consumer producer handler-fn max-retries]
  (start-consumer! consumer
    [order-events-topic]
    (fn [record]
      (let [attempts (or (get-in record [:value :_retry-count]) 0)]
        (try
          (handler-fn (:value record))
          (catch Exception e
            (if (< attempts max-retries)
              ;; Retry: republish with incremented retry count
              (send-event! producer (:topic record)
                (:key record)
                (update (:value record) :_retry-count (fnil inc 0)))
              
              ;; Max retries exceeded: send to DLQ
              (send-event! producer "dead-letter"
                (:key record)
                {:original-topic  (:topic record)
                 :original-value  (:value record)
                 :error           (.getMessage e)
                 :failed-at       (str (java.time.Instant/now))
                 :retry-count     attempts}))))))))

;; DLQ processor: manual review and replay
(defn replay-dlq-messages! [producer n]
  (let [dlq-consumer (create-consumer bootstrap-servers "dlq-processor")]
    (jc/subscribe dlq-consumer [dead-letter-topic])
    (dotimes [_ n]
      (let [[record] (jc/poll dlq-consumer 5000)]
        (when record
          (let [original (:value record)]
            ;; Republish to original topic
            (send-event! producer
              (:original-topic original)
              (:key record)
              (dissoc (:original-value original) :_retry-count))
            (jc/commit-offsets! dlq-consumer)))))))
```

---

## ขั้นตอนที่ 1835: Stream Processing ด้วย Kafka Streams

```clojure
;; Kafka Streams DSL
(ns myapp.streams
  (:import [org.apache.kafka.streams StreamsBuilder KafkaStreams]
           [org.apache.kafka.streams.kstream KStream KTable TimeWindows]))

(defn create-order-analytics [bootstrap-servers]
  (let [builder (StreamsBuilder.)]
    
    ;; Input stream
    (def order-stream
      (.stream builder "order-events"
        (Consumed/with (Serdes/String) json-serde)))
    
    ;; Filter placed orders only
    (def placed-orders
      (.filter order-stream
        (reify Predicate
          (test [_ _ v] (= "order-placed" (:type v))))))
    
    ;; Count orders per minute using windowing
    (.to
      (.count
        (.groupByKey
          (.selectKey placed-orders
            (reify KeyValueMapper
              (apply [_ _ v] (str (:user-id v)))))
          (Grouped/with (Serdes/String) json-serde))
        (TimeWindows/ofSizeWithNoGrace (Duration/ofMinutes 1)))
      "order-counts-per-minute"
      (Produced/with (WindowedSerdes/TimeWindowedSerdeFrom String) (Serdes/Long)))
    
    ;; Start streams
    (KafkaStreams.
      (.build builder)
      {"application.id"    "order-analytics"
       "bootstrap.servers" bootstrap-servers})))
```

---

## Project: Order Processing Pipeline

```clojure
(ns myapp.pipeline
  (:require [myapp.kafka :as kafka]))

;; Complete order processing pipeline with Kafka

(defn setup-pipeline! [config]
  (let [bootstrap-servers (:kafka-servers config)
        
        ;; Producers
        event-producer (create-producer bootstrap-servers)
        
        ;; Consumers (in separate threads)
        order-processor
        (future
          (start-consumer!
            (create-consumer bootstrap-servers "order-service")
            [order-events-topic]
            (fn [{:keys [value]}]
              (case (:type value)
                :order-placed
                (let [result (process-new-order! value)]
                  (send-event! event-producer "order-events"
                    (:order-id result)
                    {:type :order-confirmed :order result}))
                nil))))
        
        inventory-updater
        (future
          (start-consumer!
            (create-consumer bootstrap-servers "inventory-service")
            [order-events-topic]
            (fn [{:keys [value]}]
              (when (= :order-placed (:type value))
                (reserve-inventory! (:items value))))))
        
        notification-sender
        (future
          (start-consumer!
            (create-consumer bootstrap-servers "notification-service")
            [order-events-topic]
            (fn [{:keys [value]}]
              (case (:type value)
                :order-confirmed (send-email! (:user-email value) :confirmed value)
                :order-shipped   (send-sms!   (:user-phone value) :shipped value)
                nil))))]
    
    {:producers  {:events event-producer}
     :consumers  [order-processor inventory-updater notification-sender]
     :stop!      (fn []
                   (jc/close event-producer)
                   (doseq [c [order-processor inventory-updater notification-sender]]
                     (future-cancel c)))}))
```

---

*Part 62 จาก 100+ | ขั้นตอน 1831-1860 จาก 1000+*
