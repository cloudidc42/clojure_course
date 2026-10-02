# Part 93: Enterprise Integration Patterns
## ขั้นตอนที่ 2761-2790: Message Routing, EIP, Service Bus, Workflow Engine

---

## บทนำ

Enterprise Integration Patterns (EIP) ด้วย Clojure:
- **Message routing** - content-based, recipient-list
- **Message transformation** - enricher, filter, splitter
- **Aggregator patterns** - collect and combine
- **Workflow engine** - step-by-step process
- **Dead letter queue** - handle failures gracefully

---

## ขั้นตอนที่ 2761: Content-Based Router

```clojure
(ns myapp.enterprise.router)

;; Route messages based on content
(defprotocol MessageRouter
  (route! [this message] "Route message to appropriate handler"))

(defrecord ContentBasedRouter [routes default-handler]
  MessageRouter
  (route! [_ message]
    (let [handler (some (fn [[pred handler]]
                          (when (pred message) handler))
                         routes)]
      ((or handler default-handler) message))))

(defn make-order-router [handlers]
  (->ContentBasedRouter
    [[(fn [m] (= :premium (get-in m [:customer :tier])))
      (:premium-handler handlers)]
     [(fn [m] (> (:total m) 10000))
      (:large-order-handler handlers)]
     [(fn [m] (= :international (:shipping-type m)))
      (:international-handler handlers)]]
    (:default-handler handlers)))

;; Recipient list: send to multiple handlers
(defn recipient-list [message handlers]
  (mapv (fn [[condition handler]]
            (when (condition message)
              (handler message)))
         handlers))

;; Broadcast to all (publish-subscribe)
(defn broadcast [message handlers]
  (mapv #(% message) handlers))

;; Dynamic recipient list from database
(defn get-recipients [db message-type]
  (jdbc/execute! db
    ["SELECT handler_fn FROM routing_rules
      WHERE message_type = ? AND active = true"
     (name message-type)]))
```

---

## ขั้นตอนที่ 2762: Message Transformation Patterns

```clojure
;; Message Enricher: add data from external sources
(defn make-enricher [enrich-fn]
  (fn [message]
    (enrich-fn message)))

(defn order-enricher [db order-message]
  (let [customer (load-customer db (:customer-id order-message))
        products (load-products db (map :product-id (:items order-message)))]
    (assoc order-message
      :customer  customer
      :products  (index-by :id products))))

;; Message Filter: pass through only matching messages
(defn make-filter [predicate]
  (fn [message]
    (when (predicate message) message)))

(def payment-filter
  (make-filter #(and (= :order-placed (:type %))
                      (> (:total %) 0))))

;; Message Splitter: split one message into multiple
(defn make-splitter [split-fn]
  (fn [message]
    (split-fn message)))

(defn split-order-by-warehouse [order]
  (group-by :warehouse-id (:items order)))

(defn order-splitter [order]
  (map (fn [[warehouse-id items]]
           {:type         :warehouse-fulfillment
            :warehouse-id warehouse-id
            :order-id     (:id order)
            :items        items
            :customer     (:customer order)})
       (split-order-by-warehouse order)))

;; Message Aggregator: collect related messages
(defn make-aggregator [correlation-key-fn complete? combine-fn timeout-ms]
  (let [buckets (atom {})]
    (fn [message]
      (let [key      (correlation-key-fn message)
            bucket   (swap! buckets update key (fnil conj []) message)
            messages (get bucket key)]
        (when (complete? messages)
          (swap! buckets dissoc key)
          (combine-fn messages))))))

;; Aggregate warehouse responses into single order status
(def order-aggregator
  (make-aggregator
    :order-id
    (fn [msgs] (= (count msgs) (count-warehouses (:order-id (first msgs)))))
    (fn [msgs]
      {:type     :order-fulfillment-complete
       :order-id (:order-id (first msgs))
       :status   (if (every? #(= :fulfilled (:status %)) msgs)
                   :fulfilled :partial)
       :items    (mapcat :items msgs)})
    300000))  ; 5 min timeout
```

---

## ขั้นตอนที่ 2763: Workflow Engine

```clojure
(ns myapp.enterprise.workflow)

;; Workflow: ordered sequence of steps
(defprotocol WorkflowStep
  (step-name [this])
  (execute!  [this context])
  (can-retry? [this]))

;; Workflow executor
(defrecord WorkflowExecution [workflow-id steps state log])

(defn make-workflow [workflow-id steps]
  (->WorkflowExecution
    workflow-id
    steps
    (atom {:status   :pending
           :current  0
           :context  {}
           :started-at (java.time.Instant/now)})
    (atom [])))

(defn execute-workflow! [workflow db event-bus]
  (let [{:keys [steps state log]} workflow]
    (loop [step-idx 0 context {}]
      (if (>= step-idx (count steps))
        ;; Workflow complete
        (do
          (swap! state assoc :status :completed :context context)
          context)
        
        (let [step (nth steps step-idx)]
          (swap! log conj {:step       (step-name step)
                            :started-at (java.time.Instant/now)})
          
          (let [result
                (try
                  {:ok (execute! step context)}
                  (catch Exception e
                    {:error e}))]
            
            (if (:error result)
              (do
                (swap! state assoc
                  :status  :failed
                  :error   (str (:error result))
                  :at-step (step-name step))
                (publish! event-bus {:type       :workflow-failed
                                      :workflow-id (:workflow-id workflow)
                                      :step        (step-name step)
                                      :error       (str (:error result))})
                nil)
              
              (recur (inc step-idx)
                     (merge context (:ok result))))))))))

;; Concrete workflow steps
(defrecord ValidateOrderStep []
  WorkflowStep
  (step-name [_] "validate-order")
  (can-retry? [_] false)
  (execute! [_ {:keys [order]}]
    (let [errors (validate-order order)]
      (when (seq errors)
        (throw (ex-info "Order validation failed" {:errors errors})))
      {:validated? true})))

(defrecord ReserveInventoryStep [product-repo]
  WorkflowStep
  (step-name [_] "reserve-inventory")
  (can-retry? [_] true)
  (execute! [_ {:keys [order]}]
    (doseq [item (:items order)]
      (reserve-stock! product-repo (:product-id item) (:quantity item)))
    {:inventory-reserved? true}))

(defrecord ChargePaymentStep [payment-gateway]
  WorkflowStep
  (step-name [_] "charge-payment")
  (can-retry? [_] false)
  (execute! [_ {:keys [order customer]}]
    (let [charge (charge! payment-gateway
                          {:amount   (:total order)
                           :customer customer
                           :order-id (:id order)})]
      {:payment-id (:id charge)
       :charged?   true})))
```

---

## ขั้นตอนที่ 2764: Dead Letter Queue & Error Handling

```clojure
;; Dead Letter Queue pattern

(defrecord DeadLetterQueue [redis-pool queue-name max-retries])

(defn dlq-enqueue! [{:keys [redis-pool queue-name]} message error attempts]
  (car/wcar redis-pool
    (car/lpush (str queue-name ":dead")
      (json/encode
        {:message    message
         :error      (str error)
         :attempts   attempts
         :failed-at  (str (java.time.Instant/now))}))))

(defn dlq-process-batch! [{:keys [redis-pool queue-name max-retries]} handler n]
  (let [items (car/wcar redis-pool
                (car/lrange (str queue-name ":dead") 0 (dec n)))]
    (doseq [item-str items]
      (let [item (json/parse-string item-str true)]
        (if (< (:attempts item) max-retries)
          ;; Re-queue for retry
          (do
            (car/wcar redis-pool
              (car/lrem (str queue-name ":dead") 1 item-str)
              (car/rpush queue-name
                (json/encode (assoc (:message item)
                               :retry-count (inc (:attempts item))))))
            (println "Re-queued message for retry, attempt" (:attempts item)))
          
          ;; Max retries exceeded: alert and archive
          (do
            (send-alert! {:type    :dlq-max-retries
                          :message (:message item)
                          :error   (:error item)})
            (archive-failed-message! (:message item))))))))

;; Outbox pattern: transactional message publishing
(defn save-with-outbox! [db entity events]
  (jdbc/with-transaction [tx db]
    (save-entity! tx entity)
    (doseq [event events]
      (jdbc/execute! tx
        ["INSERT INTO outbox (event_type, payload, created_at)
          VALUES (?, ?::jsonb, NOW())"
         (name (:type event))
         (json/encode event)]))))

;; Outbox processor (run periodically)
(defn process-outbox! [db event-bus]
  (jdbc/with-transaction [tx db]
    (let [events (jdbc/execute! tx
                   ["SELECT id, event_type, payload
                     FROM outbox
                     WHERE published_at IS NULL
                     ORDER BY created_at
                     LIMIT 100
                     FOR UPDATE SKIP LOCKED"])]
      (doseq [event events]
        (publish! event-bus (json/parse-string (:outbox/payload event) true))
        (jdbc/execute! tx
          ["UPDATE outbox SET published_at = NOW() WHERE id = ?"
           (:outbox/id event)])))))
```

---

*Part 93 จาก 100+ | ขั้นตอน 2761-2790 จาก 1000+*
