# Part 32: Apache Kafka และ Event Streaming
## ขั้นตอนที่ 931-960: Producer, Consumer, Streams, Event-Driven Architecture

---

## บทนำ

Apache Kafka สำหรับ Event Streaming:
- **Kafka Producer** - ส่ง events ไปยัง topics
- **Kafka Consumer** - รับ events จาก topics
- **Kafka Streams** - process events แบบ stateful
- **Schema Registry** - ควบคุม schema ของ events
- **Exactly-once semantics** - guarantee ว่า event ถูก process 1 ครั้งเท่านั้น

---

## ขั้นตอนที่ 931: Kafka Setup

```clojure
;; deps.edn
;; {:deps {org.apache.kafka/kafka-clients {:mvn/version "3.6.0"}
;;         io.confluent/kafka-avro-serializer {:mvn/version "7.5.0"}}}

(ns myapp.kafka
  (:import [org.apache.kafka.clients.producer KafkaProducer ProducerRecord]
           [org.apache.kafka.clients.consumer KafkaConsumer ConsumerRecord]
           [org.apache.kafka.common.serialization StringSerializer StringDeserializer]))

;; Producer configuration
(def producer-config
  {"bootstrap.servers"  (or (System/getenv "KAFKA_BROKERS") "localhost:9092")
   "key.serializer"     "org.apache.kafka.common.serialization.StringSerializer"
   "value.serializer"   "org.apache.kafka.common.serialization.StringSerializer"
   "acks"               "all"              ; wait for all replicas
   "retries"            "3"
   "enable.idempotence" "true"             ; exactly-once
   "compression.type"   "lz4"})

;; Consumer configuration
(def consumer-config
  {"bootstrap.servers"  (or (System/getenv "KAFKA_BROKERS") "localhost:9092")
   "group.id"           "myapp-consumer-group"
   "key.deserializer"   "org.apache.kafka.common.serialization.StringDeserializer"
   "value.deserializer" "org.apache.kafka.common.serialization.StringDeserializer"
   "auto.offset.reset"  "earliest"
   "enable.auto.commit" "false"})           ; manual commit = at-least-once
```

---

## ขั้นตอนที่ 932: Kafka Producer

```clojure
(ns myapp.kafka.producer
  (:require [cheshire.core :as json])
  (:import [org.apache.kafka.clients.producer KafkaProducer ProducerRecord]))

;; Create producer
(defn create-producer []
  (KafkaProducer. producer-config))

;; Send event
(defn send-event! [producer topic key event]
  (let [value (json/generate-string event)
        record (if key
                 (ProducerRecord. topic key value)
                 (ProducerRecord. topic value))
        future (.send producer record)]
    ;; Synchronous confirm
    @future))

;; Async send with callback
(defn send-event-async! [producer topic key event on-success on-error]
  (let [value   (json/generate-string event)
        record  (ProducerRecord. topic key value)]
    (.send producer record
      (reify org.apache.kafka.clients.producer.Callback
        (onCompletion [_ metadata exception]
          (if exception
            (on-error exception)
            (on-success {:partition (.partition metadata)
                          :offset    (.offset metadata)})))))))

;; High-level helper
(defonce kafka-producer (atom nil))

(defn publish!
  "Publish event to Kafka topic"
  [topic event-type data]
  (let [event {:event-type event-type
                :data       data
                :timestamp  (System/currentTimeMillis)
                :id         (str (java.util.UUID/randomUUID))}]
    (send-event! @kafka-producer topic (:id event) event)
    event))

;; Usage
(comment
  (reset! kafka-producer (create-producer))
  
  (publish! "user-events" :user.created {:id 1 :name "สมชาย" :email "a@test.com"})
  (publish! "order-events" :order.placed {:order-id 100 :user-id 1 :total 5000})
  (publish! "payment-events" :payment.processed {:order-id 100 :amount 5000}))
```

---

## ขั้นตอนที่ 933: Kafka Consumer

```clojure
(ns myapp.kafka.consumer
  (:require [cheshire.core :as json]
            [clojure.tools.logging :as log])
  (:import [org.apache.kafka.clients.consumer KafkaConsumer]
           [java.time Duration]))

(defn create-consumer [group-id]
  (KafkaConsumer.
    (assoc consumer-config "group.id" group-id)))

;; Poll loop
(defn consume-loop!
  "Start consuming messages from topics"
  [consumer topics handler-fn]
  (.subscribe consumer topics)
  (future
    (try
      (loop []
        (let [records (.poll consumer (Duration/ofMillis 1000))]
          (doseq [record records]
            (try
              (let [event (json/parse-string (.value record) true)]
                (handler-fn event {:topic     (.topic record)
                                    :partition (.partition record)
                                    :offset    (.offset record)
                                    :key       (.key record)}))
              (catch Exception e
                (log/error e "Error processing message")))
            ;; Commit after each message (at-least-once)
            (.commitSync consumer)))
        (recur))
      (catch Exception e
        (log/error e "Consumer loop error"))
      (finally
        (.close consumer)))))

;; Event routing
(defmulti handle-event
  (fn [event _] (:event-type event)))

(defmethod handle-event "user.created" [event meta]
  (log/info "New user created:" (:data event))
  (send-welcome-email! (get-in event [:data :email])))

(defmethod handle-event "order.placed" [event meta]
  (log/info "New order:" (:data event))
  (process-order! (:data event)))

(defmethod handle-event :default [event meta]
  (log/warn "Unknown event type:" (:event-type event)))
```

---

## ขั้นตอนที่ 934: Consumer Groups

```clojure
;; Consumer groups = horizontal scaling
;; Partitions are distributed across group members

;; Example: 3 partitions, 3 consumers
;; Consumer 1 → Partition 0
;; Consumer 2 → Partition 1
;; Consumer 3 → Partition 2

;; Start multiple consumers for scale
(defn start-consumer-pool! [topic num-consumers handler-fn]
  (dotimes [i num-consumers]
    (let [consumer (create-consumer (str "myapp-group-" i))]
      (consume-loop! consumer [topic] handler-fn)
      (log/info (str "Started consumer " i " for " topic)))))

;; Idempotent processing (handle duplicates)
(def processed-events (atom #{}))

(defn process-idempotent! [event-id process-fn]
  (when-not (contains? @processed-events event-id)
    (swap! processed-events conj event-id)
    (process-fn)
    ;; Store in DB for persistence across restarts
    (db/mark-processed! event-id)))

(defmethod handle-event "payment.processed" [event meta]
  (process-idempotent! (:id event)
    (fn []
      (order-service/mark-paid! (get-in event [:data :order-id])))))
```

---

## ขั้นตอนที่ 935: Event Schema (Avro/JSON Schema)

```clojure
;; Event schema validation with Malli

(ns myapp.kafka.schema
  (:require [malli.core :as m]))

;; Event schemas
(def UserCreatedEvent
  [:map
   [:event-type [:= "user.created"]]
   [:id         :uuid]
   [:timestamp  :int]
   [:data [:map
           [:id    :int]
           [:name  :string]
           [:email [:re #".+@.+\..+"]]]]])

(def OrderPlacedEvent
  [:map
   [:event-type [:= "order.placed"]]
   [:id         :uuid]
   [:timestamp  :int]
   [:data [:map
           [:order-id :int]
           [:user-id  :int]
           [:items    [:vector [:map
                                [:product-id :int]
                                [:quantity   :int]
                                [:price      :double]]]]
           [:total    :double]]]])

;; Validate before publishing
(defn publish-validated!
  "Publish event with schema validation"
  [topic schema event-type data]
  (let [event {:event-type (name event-type)
                :id         (str (java.util.UUID/randomUUID))
                :timestamp  (System/currentTimeMillis)
                :data       data}]
    (when-not (m/validate schema event)
      (let [errors (m/explain schema event)]
        (throw (ex-info "Invalid event schema"
                         {:errors errors :event event}))))
    (send-event! @kafka-producer topic (:id event) event)))
```

---

## ขั้นตอนที่ 936: Event-Driven Saga Pattern

```clojure
;; Saga Pattern: distributed transactions via events
;; Order Processing Saga:
;; 1. order.placed → reserve-inventory
;; 2. inventory.reserved → charge-payment
;; 3. payment.charged → ship-order
;; 4. order.shipped → complete
;; On failure: compensating transactions

(ns myapp.sagas.order)

;; Saga state machine
(defmulti process-saga-event (fn [event _] (:event-type event)))

(defmethod process-saga-event "order.placed" [event saga-store]
  (let [order-id (get-in event [:data :order-id])]
    ;; Start saga
    (saga-store/create! saga-store
      {:saga-id  (str order-id)
       :type     :order-processing
       :state    :started
       :order-id order-id})
    
    ;; Trigger inventory reservation
    (publish! "inventory-commands"
               :inventory/reserve
               {:order-id order-id
                :items    (get-in event [:data :items])})))

(defmethod process-saga-event "inventory.reserved" [event saga-store]
  (let [order-id (get-in event [:data :order-id])]
    (saga-store/update! saga-store order-id {:state :inventory-reserved})
    
    ;; Trigger payment
    (publish! "payment-commands"
               :payment/charge
               {:order-id order-id
                :amount   (get-in event [:data :total])})))

(defmethod process-saga-event "inventory.reservation-failed" [event saga-store]
  (let [order-id (get-in event [:data :order-id])]
    (saga-store/update! saga-store order-id {:state :failed})
    
    ;; Compensating transaction: cancel order
    (publish! "order-commands"
               :order/cancel
               {:order-id order-id
                :reason   :inventory-unavailable})))

(defmethod process-saga-event "payment.charged" [event saga-store]
  (let [order-id (get-in event [:data :order-id])]
    (saga-store/update! saga-store order-id {:state :payment-completed})
    
    ;; Ship order
    (publish! "shipping-commands"
               :shipping/dispatch
               {:order-id order-id})))
```

---

## Project: Real-Time Order Processing System

```clojure
(ns shop.order-processor
  (:require [myapp.kafka.consumer :as consumer]
            [myapp.kafka.producer :as producer]))

;; Event bus
(def event-handlers
  {"order.placed"         handle-order-placed
   "payment.completed"    handle-payment-completed
   "inventory.low"        handle-low-inventory
   "order.cancelled"      handle-order-cancelled})

(defn start-order-processor! []
  (let [consumer (consumer/create-consumer "order-processor")]
    (consumer/consume-loop!
      consumer
      ["order-events" "payment-events" "inventory-events"]
      (fn [event meta]
        (if-let [handler (get event-handlers (:event-type event))]
          (do
            (log/info "Processing:" (:event-type event))
            (handler event))
          (log/warn "No handler for:" (:event-type event)))))))

;; Analytics consumer
(defn start-analytics-consumer! []
  (let [consumer (consumer/create-consumer "analytics-tracker")]
    (consumer/consume-loop!
      consumer
      ["order-events" "user-events"]
      (fn [event _]
        (analytics/track! {:event-type (:event-type event)
                            :timestamp  (:timestamp event)
                            :data       (:data event)})))))

;; Run system
(defn -main []
  (reset! kafka-producer (producer/create-producer))
  (start-order-processor!)
  (start-analytics-consumer!)
  (println "Order processing system started"))
```

---

*Part 32 จาก 100+ | ขั้นตอน 931-960 จาก 1000+*
