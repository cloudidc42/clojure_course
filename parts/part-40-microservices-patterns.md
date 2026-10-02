# Part 40: Microservices Patterns
## ขั้นตอนที่ 1171-1200: Service Discovery, API Gateway, Circuit Breaker, Saga

---

## บทนำ

Microservices architecture ด้วย Clojure:
- **Service Discovery** - Consul/etcd integration
- **API Gateway** - rate limiting, authentication, routing
- **Circuit Breaker** - fault tolerance pattern
- **Saga Pattern** - distributed transactions
- **CQRS** - Command Query Responsibility Segregation

---

## ขั้นตอนที่ 1171: Service Discovery ด้วย Consul

```clojure
;; deps.edn
;; {:deps {com.ecwid.consul/consul-api {:mvn/version "1.4.5"}}}

(ns myapp.discovery
  (:import [com.ecwid.consul.v1 ConsulClient]
           [com.ecwid.consul.v1.agent.model NewService]))

(def consul (ConsulClient. "consul" 8500))

;; Register service with Consul
(defn register-service! [{:keys [id name host port health-path tags]}]
  (let [service (doto (NewService.)
                  (.setId id)
                  (.setName name)
                  (.setAddress host)
                  (.setPort port)
                  (.setTags (java.util.ArrayList. tags)))]
    
    ;; Add health check
    (let [check (doto (NewService$NewServiceCheck.)
                  (.setHttp (str "http://" host ":" port health-path))
                  (.setInterval "10s")
                  (.setTimeout "5s")
                  (.setDeregisterCriticalServiceAfter "30m"))]
      (.setCheck service check))
    
    (.agentServiceRegister consul service)
    (println "Registered service:" id)))

;; Discover healthy instances
(defn discover-service [service-name]
  (let [result (.getHealthServices consul service-name true nil)]
    (->> (.getValue result)
         (map (fn [entry]
                (let [svc (.getService entry)]
                  {:id      (.getId svc)
                   :host    (.getAddress svc)
                   :port    (.getPort svc)
                   :tags    (into [] (.getTags svc))}))))))

;; Load balancer (round-robin)
(def service-cursors (atom {}))

(defn round-robin-select [instances service-name]
  (let [n    (count instances)
        idx  (get (swap! service-cursors update service-name
                          (fnil #(mod (inc %) n) 0))
                   service-name)]
    (when (pos? n)
      (nth instances idx))))

(defn get-service-url [service-name]
  (when-let [instance (-> (discover-service service-name)
                           (round-robin-select service-name))]
    (str "http://" (:host instance) ":" (:port instance))))
```

---

## ขั้นตอนที่ 1172: API Gateway Pattern

```clojure
(ns gateway.core
  (:require [reitit.ring :as ring]
            [taoensso.carmine :as car]))

;; Rate limiter ด้วย Redis (sliding window)
(defn rate-limit!
  "Returns true if request is allowed, false if rate limited"
  [redis-conn key limit window-seconds]
  (car/wcar redis-conn
    (let [now      (System/currentTimeMillis)
          window   (* window-seconds 1000)
          min-time (- now window)]
      (car/multi-exec
        (car/zremrangebyscore key "-inf" min-time)
        (car/zadd key now now)
        (car/zcard key)
        (car/expire key window-seconds))
      (let [count (nth (car/wcar redis-conn (car/zcard key)) 0)]
        (<= count limit)))))

;; Auth middleware
(defn wrap-gateway-auth [handler jwt-secret]
  (fn [request]
    (let [token (some-> (get-in request [:headers "authorization"])
                         (clojure.string/replace #"^Bearer " ""))]
      (if token
        (try
          (let [claims (jwt/unsign token jwt-secret {:alg :hs256})]
            (handler (assoc request :auth-claims claims)))
          (catch Exception _
            {:status 401 :body {:error "Invalid token"}}))
        {:status 401 :body {:error "Missing token"}}))))

;; Proxy to upstream services
(defn proxy-to-service [service-name path-prefix]
  (fn [request]
    (if-let [base-url (discovery/get-service-url service-name)]
      (let [upstream-path (clojure.string/replace
                            (:uri request)
                            (re-pattern (str "^" path-prefix))
                            "")
            resp @(org.httpkit.client/request
                    {:url     (str base-url upstream-path)
                     :method  (:request-method request)
                     :headers (dissoc (:headers request) "host")
                     :body    (when (:body request)
                                (slurp (:body request)))})]
        {:status  (:status resp)
         :headers (:headers resp)
         :body    (:body resp)})
      {:status 503 :body {:error (str service-name " service unavailable")}})))

;; Gateway router
(defn create-gateway [redis-conn]
  (ring/ring-handler
    (ring/router
      [["/api/users/*path"     {:handler (proxy-to-service "user-service"    "/api/users")}]
       ["/api/products/*path"  {:handler (proxy-to-service "product-service" "/api/products")}]
       ["/api/orders/*path"    {:handler (proxy-to-service "order-service"   "/api/orders")}]]
      {:data {:middleware [(wrap-gateway-auth jwt-secret)
                             (wrap-rate-limit redis-conn 100 60)]}})))
```

---

## ขั้นตอนที่ 1173: Circuit Breaker ที่สมบูรณ์

```clojure
(ns myapp.circuit-breaker)

;; Circuit states: :closed, :open, :half-open
(defrecord CircuitBreaker
  [name
   state           ; atom: {:state :closed :failures 0 :last-failure nil}
   failure-threshold
   timeout-ms
   success-threshold])

(defn create-breaker
  [name & {:keys [failure-threshold timeout-ms success-threshold]
           :or   {failure-threshold 5
                  timeout-ms        30000
                  success-threshold 2}}]
  (->CircuitBreaker
    name
    (atom {:state    :closed
            :failures 0
            :successes 0
            :last-failure nil})
    failure-threshold
    timeout-ms
    success-threshold))

(defn- state-allowed? [{:keys [state last-failure]} timeout-ms]
  (case state
    :closed    true
    :open      (and last-failure
                    (> (- (System/currentTimeMillis) last-failure)
                       timeout-ms))
    :half-open true))

(defn- record-success! [breaker-state threshold]
  (case (:state breaker-state)
    :half-open (if (>= (inc (:successes breaker-state)) threshold)
                 {:state :closed :failures 0 :successes 0 :last-failure nil}
                 (update breaker-state :successes inc))
    :closed    (assoc breaker-state :failures 0)
    breaker-state))

(defn- record-failure! [breaker-state threshold]
  (let [failures (inc (:failures breaker-state))]
    (if (>= failures threshold)
      {:state        :open
       :failures     failures
       :successes    0
       :last-failure (System/currentTimeMillis)}
      (assoc breaker-state :failures failures))))

(defn call!
  "Execute f through circuit breaker. Returns {:ok result} or {:error msg}"
  [breaker f]
  (let [current @(:state breaker)]
    (if-not (state-allowed? current (:timeout-ms breaker))
      {:error "Circuit breaker is OPEN" :state :open}
      (do
        ;; Transition to half-open if we're retrying after open
        (when (= :open (:state current))
          (swap! (:state breaker) assoc :state :half-open))
        
        (try
          (let [result (f)]
            (swap! (:state breaker) record-success! (:success-threshold breaker))
            {:ok result})
          (catch Exception e
            (swap! (:state breaker) record-failure! (:failure-threshold breaker))
            {:error (.getMessage e)}))))))

;; Macro for convenience
(defmacro with-circuit-breaker [breaker & body]
  `(let [result# (call! ~breaker (fn [] ~@body))]
     (if (:ok result#)
       (:ok result#)
       (throw (ex-info (:error result#) {:circuit-breaker (:name ~breaker)})))))

;; Example
(def payment-breaker
  (create-breaker "payment-service"
    :failure-threshold 3
    :timeout-ms        10000))

(defn charge-customer! [amount customer-id]
  (with-circuit-breaker payment-breaker
    (payment-api/charge! amount customer-id)))
```

---

## ขั้นตอนที่ 1174: CQRS Pattern

```clojure
(ns myapp.cqrs)

;; ===== Commands (write side) =====

(defprotocol Command
  (validate [cmd])
  (execute! [cmd context]))

(defrecord PlaceOrderCommand [user-id items shipping-address]
  Command
  
  (validate [cmd]
    (cond
      (empty? items)           {:error "Order must have items"}
      (nil? shipping-address)  {:error "Shipping address required"}
      :else                    nil))
  
  (execute! [cmd {:keys [db event-bus]}]
    (let [order (order-domain/create-order
                  (java.util.UUID/randomUUID)
                  user-id
                  items
                  shipping-address)]
      (order-repo/save! db order)
      (event-bus/publish! event-bus {:type :order-placed :order order})
      {:order-id (:id order)})))

;; Command bus
(defn dispatch-command! [bus command context]
  (if-let [err (validate command)]
    {:error err}
    (execute! command context)))

;; ===== Queries (read side) =====

;; Separate read models optimized for queries
(defn get-order-summary
  "Denormalized view optimized for display"
  [db order-id]
  (jdbc/execute-one! db
    ["SELECT o.id, o.status, o.total,
             u.name as customer_name, u.email as customer_email,
             COUNT(oi.id) as item_count
      FROM orders o
      JOIN users u ON o.user_id = u.id
      JOIN order_items oi ON oi.order_id = o.id
      WHERE o.id = ?
      GROUP BY o.id, u.name, u.email"
     order-id]))

(defn get-user-order-history
  "Paginated order list for user"
  [db user-id {:keys [page limit status]}]
  (let [offset (* (dec (or page 1)) (or limit 20))]
    (jdbc/execute! db
      [(cond-> "SELECT id, status, total, created_at
                FROM orders WHERE user_id = ?"
         status (str " AND status = ?"))
       user-id
       (when status status)])))

;; Event projector: update read models from events
(defmulti project-event! (fn [db event] (:type event)))

(defmethod project-event! :order-placed [db {:keys [order]}]
  (jdbc/execute! db
    ["INSERT INTO order_summaries (id, user_id, status, total, item_count, created_at)
      VALUES (?, ?, ?, ?, ?, ?)"
     (:id order) (:user-id order) "placed"
     (:total order) (count (:items order))
     (java.time.Instant/now)]))

(defmethod project-event! :order-shipped [db {:keys [order-id tracking]}]
  (jdbc/execute! db
    ["UPDATE order_summaries SET status = 'shipped', tracking_number = ?
      WHERE id = ?" tracking order-id]))
```

---

## ขั้นตอนที่ 1175: Saga Pattern สำหรับ Distributed Transactions

```clojure
(ns myapp.saga)

;; Saga: sequence of local transactions with compensating actions

;; Saga step structure
(defrecord SagaStep
  [name
   execute-fn      ; (ctx) -> result
   compensate-fn]) ; (ctx) -> nil, undo the execute

(defn create-step [name execute-fn compensate-fn]
  (->SagaStep name execute-fn compensate-fn))

;; Saga executor
(defn execute-saga! [steps initial-ctx]
  (loop [remaining steps
         completed  []
         ctx        initial-ctx]
    (if-let [step (first remaining)]
      (try
        (let [result ((:execute-fn step) ctx)
              new-ctx (merge ctx result)]
          (recur (rest remaining)
                 (conj completed step)
                 new-ctx))
        (catch Exception e
          ;; Compensate all completed steps in reverse
          (doseq [completed-step (reverse completed)]
            (try
              ((:compensate-fn completed-step) ctx)
              (catch Exception ce
                (println "Compensation failed for" (:name completed-step)
                          (.getMessage ce)))))
          {:error (.getMessage e) :failed-at (:name step)}))
      {:success true :context ctx})))

;; Order Placement Saga
(def order-saga
  [(create-step
     "reserve-inventory"
     (fn [ctx]
       (let [reservation (inventory/reserve! (:items ctx))]
         {:reservation-id (:id reservation)}))
     (fn [ctx]
       (inventory/cancel-reservation! (:reservation-id ctx))))
   
   (create-step
     "charge-payment"
     (fn [ctx]
       (let [charge (payment/charge! (:total ctx) (:user-id ctx))]
         {:payment-id (:id charge)}))
     (fn [ctx]
       (payment/refund! (:payment-id ctx))))
   
   (create-step
     "create-order"
     (fn [ctx]
       (let [order (order/create! ctx)]
         {:order-id (:id order)}))
     (fn [ctx]
       (order/cancel! (:order-id ctx))))
   
   (create-step
     "send-confirmation"
     (fn [ctx]
       (email/send-order-confirmation! (:user-id ctx) (:order-id ctx))
       {})
     (fn [_] nil))])  ; email sent, can't unsend

(defn place-order! [user-id items]
  (execute-saga! order-saga
    {:user-id user-id
     :items   items
     :total   (calculate-total items)}))
```

---

## ขั้นตอนที่ 1176: Event-Driven Communication

```clojure
(ns myapp.events)

;; Event bus abstraction
(defprotocol EventBus
  (publish!    [bus event])
  (subscribe!  [bus event-type handler])
  (unsubscribe! [bus event-type handler]))

;; In-process event bus (for testing/monolith)
(defrecord LocalEventBus [handlers]
  EventBus
  
  (publish! [_ event]
    (doseq [handler (get @handlers (:type event) [])]
      (try
        (handler event)
        (catch Exception e
          (println "Event handler error:" (.getMessage e))))))
  
  (subscribe! [_ event-type handler]
    (swap! handlers update event-type (fnil conj []) handler))
  
  (unsubscribe! [_ event-type handler]
    (swap! handlers update event-type
           (fnil #(remove #{handler} %) []))))

;; Kafka-backed event bus
(defrecord KafkaEventBus [producer consumer-group]
  EventBus
  
  (publish! [_ event]
    (let [topic  (name (:type event))
          key    (str (:aggregate-id event))
          value  (cheshire.core/generate-string event)]
      (.send producer (org.apache.kafka.clients.producer.ProducerRecord. topic key value))))
  
  (subscribe! [_ event-type handler]
    ;; Returns consumer future
    (kafka/start-consumer! consumer-group [(name event-type)] handler)))

;; Event handlers
(defn handle-order-placed! [event]
  (let [{:keys [order-id user-id items total]} (:data event)]
    (println "Order placed:" order-id "for user" user-id "total:" total)
    ;; Update inventory
    (inventory/reduce-stock! items)
    ;; Send notifications
    (notification/send! user-id {:type :order-confirmed :order-id order-id})))
```

---

## Project: Order Processing Microservices

```clojure
;; Three services communicating via Kafka

;; 1. Order Service
(ns order.service)

(defn handle-place-order [request]
  (let [order (order/create! (:body-params request))]
    (event-bus/publish! bus {:type         :order-placed
                               :aggregate-id (:id order)
                               :data         order})
    {:status 201 :body order}))

;; 2. Inventory Service (listens to order events)
(ns inventory.consumer)

(kafka/subscribe!
  [:order-placed]
  (fn [event]
    (inventory/reserve-for-order!
      (:aggregate-id event)
      (get-in event [:data :items]))))

;; 3. Notification Service (listens to both)
(ns notification.consumer)

(kafka/subscribe!
  [:order-placed :order-shipped :order-cancelled]
  (fn [event]
    (let [template (case (:type event)
                      :order-placed    :order-confirmation
                      :order-shipped   :shipping-notification
                      :order-cancelled :cancellation-notice)]
      (email/send-templated!
        (get-in event [:data :user-email])
        template
        (:data event)))))
```

---

*Part 40 จาก 100+ | ขั้นตอน 1171-1200 จาก 1000+*
