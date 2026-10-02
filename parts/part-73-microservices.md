# Part 73: Microservices Architecture
## ขั้นตอนที่ 2161-2190: Service Discovery, Circuit Breaker, API Gateway, gRPC

---

## บทนำ

Microservices patterns ใน Clojure:
- **Service registry** - discover services dynamically
- **Circuit breaker** - fail fast, prevent cascade
- **API Gateway** - single entry point
- **gRPC** - high-performance inter-service communication
- **Saga pattern** - distributed transactions

---

## ขั้นตอนที่ 2161: Circuit Breaker

```clojure
(ns myapp.resilience.circuit-breaker)

;; Circuit breaker states: :closed -> :open -> :half-open -> :closed
(defrecord CircuitBreaker
  [name state failure-count last-failure-time
   failure-threshold recovery-timeout-ms])

(defn circuit-breaker
  [name & {:keys [failure-threshold recovery-timeout-ms]
           :or   {failure-threshold    5
                  recovery-timeout-ms  60000}}]
  (atom (->CircuitBreaker name :closed 0 nil
                            failure-threshold recovery-timeout-ms)))

(defn circuit-open? [cb]
  (let [{:keys [state last-failure-time recovery-timeout-ms]} @cb]
    (case state
      :closed false
      :open   (if (and last-failure-time
                        (> (- (System/currentTimeMillis) last-failure-time)
                            recovery-timeout-ms))
                (do
                  (swap! cb assoc :state :half-open)
                  false)
                true)
      :half-open false)))

(defn record-success! [cb]
  (swap! cb assoc :state :closed :failure-count 0 :last-failure-time nil))

(defn record-failure! [cb]
  (swap! cb
    (fn [{:keys [failure-count failure-threshold] :as state}]
      (let [new-count (inc failure-count)]
        (if (>= new-count failure-threshold)
          (assoc state
            :state :open
            :failure-count new-count
            :last-failure-time (System/currentTimeMillis))
          (assoc state :failure-count new-count))))))

(defmacro with-circuit-breaker [cb fallback & body]
  `(if (circuit-open? ~cb)
     (do
       (println "Circuit OPEN for" (:name @~cb))
       ~fallback)
     (try
       (let [result# (do ~@body)]
         (record-success! ~cb)
         result#)
       (catch Exception e#
         (record-failure! ~cb)
         (if (circuit-open? ~cb)
           (do (println "Circuit OPENED after failure") ~fallback)
           (throw e#))))))

;; Usage
(def payment-cb (circuit-breaker "payment-service"
                   :failure-threshold 3
                   :recovery-timeout-ms 30000))

(defn charge-customer! [amount]
  (with-circuit-breaker payment-cb
    {:status :service-unavailable :retry-after 30}
    (call-payment-service! amount)))
```

---

## ขั้นตอนที่ 2162: Retry with Exponential Backoff

```clojure
(ns myapp.resilience.retry)

(defn with-retry
  [f {:keys [max-attempts backoff-ms jitter max-backoff-ms]
      :or   {max-attempts  5
             backoff-ms    100
             jitter        true
             max-backoff-ms 30000}}]
  (loop [attempt 1]
    (let [result (try {:ok (f)}
                       (catch Exception e
                         {:error e
                          :retryable? (retryable? e)}))]
      (cond
        (:ok result) (:ok result)
        
        (not (:retryable? result))
        (throw (:error result))
        
        (>= attempt max-attempts)
        (throw (ex-info "Max retries exceeded"
                         {:cause (:error result)
                          :attempts attempt}))
        
        :else
        (let [delay    (min (* backoff-ms (Math/pow 2 (dec attempt)))
                             max-backoff-ms)
               jitter-ms (when jitter (rand-int (/ delay 2)))]
          (Thread/sleep (long (+ delay (or jitter-ms 0))))
          (recur (inc attempt)))))))

(defn retryable? [e]
  (or
    (instance? java.net.SocketTimeoutException e)
    (instance? java.net.ConnectException e)
    (when-let [data (ex-data e)]
      (#{503 429 502 504} (:status data)))))
```

---

## ขั้นตอนที่ 2163: Service Discovery

```clojure
(ns myapp.service-discovery
  (:require [etcd-clj.core :as etcd]))

;; Simple service registry using etcd or Redis
(defprotocol ServiceRegistry
  (register! [this service-name instance])
  (deregister! [this service-name instance-id])
  (discover [this service-name])
  (watch-service [this service-name callback]))

;; Redis-based registry
(defrecord RedisServiceRegistry [redis-pool]
  ServiceRegistry
  
  (register! [_ service-name {:keys [id host port metadata ttl-seconds]}]
    (let [key   (str "services:" service-name ":" id)
          value (json/generate-string
                  {:id host port metadata
                   :registered-at (System/currentTimeMillis)})]
      (car/wcar redis-pool
        (car/set key value)
        (car/expire key (or ttl-seconds 30)))))
  
  (deregister! [_ service-name instance-id]
    (car/wcar redis-pool
      (car/del (str "services:" service-name ":" instance-id))))
  
  (discover [_ service-name]
    (let [pattern (str "services:" service-name ":*")
          keys    (car/wcar redis-pool (car/keys pattern))]
      (when (seq keys)
        (->> (car/wcar redis-pool
               (apply car/mget keys))
             (filter identity)
             (map #(json/parse-string % true))))))
  
  (watch-service [_ service-name callback]
    ;; Poll for changes (simple implementation)
    (future
      (loop [prev-instances nil]
        (Thread/sleep 5000)
        (let [current (discover _ service-name)]
          (when (not= current prev-instances)
            (callback current))
          (recur current))))))

;; Load balancer
(defn round-robin-load-balancer [registry service-name]
  (let [counter (atom 0)]
    (fn []
      (let [instances (discover registry service-name)]
        (when (seq instances)
          (let [idx (mod (swap! counter inc) (count instances))]
            (nth instances idx)))))))

;; HTTP client with service discovery
(defn call-service! [lb-fn path & {:keys [method body]}]
  (if-let [instance (lb-fn)]
    (http/request
      {:method  (or method :get)
       :url     (str "http://" (:host instance) ":" (:port instance) path)
       :body    (when body (json/generate-string body))
       :headers {"Content-Type" "application/json"}})
    (throw (ex-info "No instances available" {:path path}))))
```

---

## ขั้นตอนที่ 2164: Saga Pattern for Distributed Transactions

```clojure
(ns myapp.saga)

;; Saga: sequence of local transactions with compensations
(defrecord SagaStep [name execute compensate])

(defn saga-step [name execute compensate]
  (->SagaStep name execute compensate))

(defn execute-saga! [steps context]
  (loop [steps     steps
         completed []
         ctx       context]
    (if (empty? steps)
      {:success true :context ctx}
      (let [step   (first steps)
            result (try
                     {:ok ((:execute step) ctx)}
                     (catch Exception e
                       {:error e}))]
        (if (:ok result)
          (recur (rest steps)
                 (conj completed step)
                 (merge ctx (:ok result)))
          
          ;; Compensate all completed steps in reverse
          (do
            (doseq [completed-step (reverse completed)]
              (try
                ((:compensate completed-step) ctx)
                (catch Exception ce
                  (println "Compensation failed for" (:name completed-step)
                           (.getMessage ce)))))
            {:success false
             :error   (:error result)
             :compensated (map :name completed)}))))))

;; Order saga example
(def checkout-saga
  [(saga-step
     :reserve-inventory
     (fn [ctx] {:reservation-id (reserve-items! (:items ctx))})
     (fn [ctx] (release-reservation! (:reservation-id ctx))))
   
   (saga-step
     :charge-payment
     (fn [ctx] {:payment-id (charge-payment! (:amount ctx) (:payment-method ctx))})
     (fn [ctx] (refund-payment! (:payment-id ctx))))
   
   (saga-step
     :create-order
     (fn [ctx] {:order-id (create-order! ctx)})
     (fn [ctx] (cancel-order! (:order-id ctx))))
   
   (saga-step
     :send-confirmation
     (fn [ctx] (send-order-confirmation! (:order-id ctx) (:email ctx)))
     (fn [_] nil))])  ; Email can't be unsent

(defn process-checkout! [cart payment-method customer]
  (execute-saga! checkout-saga
    {:items          (:items cart)
     :amount         (:total cart)
     :payment-method payment-method
     :email          (:email customer)}))
```

---

## ขั้นตอนที่ 2165: API Gateway

```clojure
(ns myapp.gateway)

;; Reverse proxy with load balancing and auth
(defn proxy-request! [target-url request opts]
  (let [headers (-> (:headers request)
                     (dissoc "host")
                     (assoc "X-Forwarded-For" (get-in request [:remote-addr])
                             "X-Forwarded-Host" (get-in request [:headers "host"])))
        response (http/request
                   {:method  (:request-method request)
                    :url     (str target-url (:uri request)
                                   (when-let [q (:query-string request)]
                                     (str "?" q)))
                    :headers headers
                    :body    (:body request)
                    :timeout (:timeout opts 10000)})]
    response))

;; Gateway routing table
(def routes
  [{:path-prefix "/api/products"
    :service     :product-service
    :middleware  [:auth :rate-limit :cache]}
   
   {:path-prefix "/api/orders"
    :service     :order-service
    :middleware  [:auth :rate-limit]}
   
   {:path-prefix "/api/payments"
    :service     :payment-service
    :middleware  [:auth :rate-limit :audit-log]}])

(defn gateway-handler [registry circuit-breakers request]
  (let [path     (:uri request)
        route    (first (filter #(.startsWith path (:path-prefix %)) routes))]
    (if-not route
      {:status 404 :body {:error "Not found"}}
      
      (let [service  (:service route)
            lb       (round-robin-load-balancer registry service)
            cb       (get circuit-breakers service)
            instance (lb)]
        
        (if-not instance
          {:status 503 :body {:error "Service unavailable"}}
          
          (with-circuit-breaker cb
            {:status 503 :body {:error "Circuit open"}}
            (proxy-request! (str "http://" (:host instance) ":" (:port instance))
                             request {:timeout 10000})))))))
```

---

*Part 73 จาก 100+ | ขั้นตอน 2161-2190 จาก 1000+*
