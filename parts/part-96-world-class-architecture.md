# Part 96: World-Class Architecture Patterns
## ขั้นตอนที่ 2851-2880: Hexagonal Architecture, Ports & Adapters, SOLID in FP

---

## บทนำ

Architecture patterns ระดับ world-class:
- **Hexagonal Architecture** - ports & adapters
- **SOLID principles in FP** - functional equivalents
- **Effect systems** - managing side effects explicitly
- **Dependency injection via functions** - no framework needed
- **Architecture fitness functions** - automated quality gates

---

## ขั้นตอนที่ 2851: Hexagonal Architecture ใน Clojure

```clojure
(ns myapp.architecture)

;; Hexagonal Architecture: business logic at center
;; Ports: interfaces defining what the core needs
;; Adapters: concrete implementations of those interfaces

;; ── PORTS (defined in domain/application layer) ──

;; Primary port: what the application exposes
(defprotocol OrderApplicationPort
  (place-order [this command])
  (cancel-order [this order-id reason])
  (get-order [this order-id]))

;; Secondary ports: what the application needs
(defprotocol OrderPersistencePort
  (save [this order])
  (find-by-id [this id])
  (find-by-customer [this customer-id]))

(defprotocol PaymentPort
  (charge [this amount customer payment-method])
  (refund [this payment-id amount]))

(defprotocol NotificationPort
  (notify-order-placed [this order customer])
  (notify-order-cancelled [this order customer reason]))

;; ── DOMAIN CORE (pure logic, no I/O) ──

(defn process-place-order-command
  "Pure function: returns domain events and new state"
  [command existing-orders inventory]
  (let [errors (validate-place-order-command command existing-orders inventory)]
    (if (seq errors)
      {:error errors}
      {:events [{:type :order-placed :data (build-order command)}]
       :state  (build-order command)})))

;; ── APPLICATION SERVICE ──

(defrecord OrderService [persistence-port payment-port notification-port]
  OrderApplicationPort
  
  (place-order [_ command]
    ;; Orchestrate ports to fulfill use case
    (let [customer  (find-by-id persistence-port (:customer-id command))
          result    (process-place-order-command command [] {})
          order     (:state result)]
      
      (when (:error result)
        (throw (ex-info "Order invalid" {:errors (:error result)})))
      
      (save persistence-port order)
      
      (when (:payment command)
        (charge payment-port
                (:total order)
                customer
                (:payment command)))
      
      (notify-order-placed notification-port order customer)
      order))
  
  (cancel-order [_ order-id reason]
    (let [order    (find-by-id persistence-port order-id)
          cancelled (-> order (assoc :status :cancelled :cancel-reason reason))]
      (save persistence-port cancelled)
      cancelled))
  
  (get-order [_ order-id]
    (find-by-id persistence-port order-id)))

;; ── ADAPTERS (infrastructure layer) ──

;; Database adapter
(defrecord PostgresOrderRepository [db]
  OrderPersistencePort
  (save [_ order] (upsert-order! db order))
  (find-by-id [_ id] (find-order-by-id db id))
  (find-by-customer [_ cid] (find-orders-by-customer db cid)))

;; In-memory adapter (for tests)
(defrecord InMemoryOrderRepository [store]
  OrderPersistencePort
  (save [_ order] (swap! store assoc (:id order) order))
  (find-by-id [_ id] (get @store id))
  (find-by-customer [_ cid] (filter #(= cid (:customer-id %)) (vals @store))))

;; Stripe payment adapter
(defrecord StripePaymentAdapter [api-key]
  PaymentPort
  (charge [_ amount customer payment-method]
    (stripe/create-payment-intent
      {:amount   (* 100 (:amount amount))
       :currency (name (:currency amount))
       :customer (:stripe-customer-id customer)}))
  (refund [_ payment-id amount]
    (stripe/create-refund {:payment_intent payment-id})))

;; Test double for payment
(defrecord FakePaymentAdapter [charges refunds]
  PaymentPort
  (charge [_ amount customer _]
    (let [id (str (java.util.UUID/randomUUID))]
      (swap! charges conj {:id id :amount amount :customer customer})
      {:id id :status :succeeded}))
  (refund [_ payment-id amount]
    (swap! refunds conj {:payment-id payment-id :amount amount})))
```

---

## ขั้นตอนที่ 2852: Effect Systems

```clojure
;; Effects: model side effects as data, interpret separately

;; Effect types (data, not functions!)
(defrecord DbRead  [query params])
(defrecord DbWrite [sql params])
(defrecord HttpGet [url headers])
(defrecord SendEmail [to subject body])
(defrecord Log     [level message data])

;; Programs return sequences of effects + result
(defn place-order-program [order]
  ;; Returns [effects result] — no actual I/O
  (let [validate-effect (->DbRead "SELECT * FROM products WHERE id = ANY(?)"
                                   [(:product-ids order)])
        log-effect      (->Log :info "Processing order" {:order-id (:id order)})]
    [[log-effect validate-effect]  ; effects to run
     {:status :pending}]))          ; pure result

;; Interpreter: actually performs effects
(defmulti interpret-effect!
  (fn [_ effect] (type effect)))

(defmethod interpret-effect! DbRead
  [context {:keys [query params]}]
  (jdbc/execute! (:db context) (into [query] params)))

(defmethod interpret-effect! DbWrite
  [context {:keys [sql params]}]
  (jdbc/execute! (:db context) (into [sql] params)))

(defmethod interpret-effect! HttpGet
  [_ {:keys [url headers]}]
  (http/get url {:headers headers}))

(defmethod interpret-effect! SendEmail
  [context {:keys [to subject body]}]
  (email/send! (:email-client context) to subject body))

(defmethod interpret-effect! Log
  [_ {:keys [level message data]}]
  (log/log level message data))

;; Run program with effects
(defn run-program! [context program-fn & args]
  (let [[effects result] (apply program-fn args)]
    (doseq [effect effects]
      (interpret-effect! context effect))
    result))

;; Test without real I/O
(defn test-place-order []
  (let [recorded (atom [])
        mock-ctx {:record! (fn [e] (swap! recorded conj e))}]
    (run-program! mock-ctx place-order-program test-order)
    @recorded))  ; Check what effects were requested
```

---

## ขั้นตอนที่ 2853: Architecture Fitness Functions

```clojure
;; Automated architecture tests: enforce architectural rules

(ns myapp.arch-test
  (:require [clojure.test :refer [deftest is testing]]))

;; Rule: domain layer must not import infrastructure
(deftest no-infra-in-domain
  (let [domain-nses (find-namespaces-matching #"myapp\.domain\.*")
        infra-nses  #{"myapp.infrastructure" "next.jdbc" "carmine"}]
    (doseq [ns domain-nses]
      (let [imports (ns-requires ns)]
        (is (empty? (clojure.set/intersection imports infra-nses))
            (str "Domain ns " ns " imports infrastructure: "
                 (clojure.set/intersection imports infra-nses)))))))

;; Rule: all public functions in domain must have specs
(deftest all-domain-fns-have-specs
  (let [domain-vars (ns-publics 'myapp.domain.order)]
    (doseq [[sym var] domain-vars]
      (when (fn? @var)
        (is (s/get-spec (keyword "myapp.domain.order" (name sym)))
            (str "Missing spec for " sym))))))

;; Rule: no direct DB calls from handlers
(deftest handlers-use-services
  (let [handler-nses (find-namespaces-matching #"myapp\.interface\.http\.*")]
    (doseq [ns handler-nses]
      (doseq [[sym var] (ns-publics ns)]
        (let [body (source @var)]
          (is (not (re-find #"jdbc|execute!|select\s" body))
              (str "Handler " sym " makes direct DB calls")))))))

;; Rule: all API endpoints must be documented
(deftest all-endpoints-documented
  (let [routes (get-all-routes)]
    (doseq [route routes]
      (is (:doc (meta (:handler route)))
          (str "Undocumented endpoint: " (:path route) " " (:method route))))))

;; Dependency graph analysis
(defn check-no-cycles [namespaces]
  (let [deps (into {} (map (fn [ns]
                              [ns (set (ns-requires ns))])
                            namespaces))]
    (loop [visited #{} stack [] current (first namespaces)]
      ;; DFS for cycle detection
      ...)))
```

---

## ขั้นตอนที่ 2854: Algebraic Effects Pattern

```clojure
;; Model effects algebraically for testability

;; Effect algebra: effects as values
(defprotocol Effect
  (effect-type [this])
  (effect-data [this]))

;; Algebraic effects handler
(defmacro with-effects [bindings & body]
  `(binding ~bindings
     ~@body))

;; Default (production) effect handlers
(def ^:dynamic *db-query*
  (fn [sql params]
    (jdbc/execute! (current-db) (into [sql] params))))

(def ^:dynamic *http-get*
  (fn [url opts]
    (http/get url opts)))

(def ^:dynamic *send-notification*
  (fn [user msg]
    (notification/send! user msg)))

;; Business logic uses dynamic vars (not direct calls)
(defn get-user-with-orders [user-id]
  (let [user   (*db-query* "SELECT * FROM users WHERE id = ?" [user-id])
        orders (*db-query* "SELECT * FROM orders WHERE user_id = ?" [user-id])]
    (assoc user :orders orders)))

;; Test with fake effects
(deftest test-get-user-with-orders
  (let [fake-db {"SELECT * FROM users WHERE id = ?"
                  [{:id 1 :name "Alice"}]
                  "SELECT * FROM orders WHERE user_id = ?"
                  [{:id 10 :total 100}]}]
    (with-effects [*db-query* (fn [sql params]
                                 (get fake-db sql []))]
      (let [result (get-user-with-orders 1)]
        (is (= "Alice" (:name result)))
        (is (= 1 (count (:orders result))))))))
```

---

*Part 96 จาก 100+ | ขั้นตอน 2851-2880 จาก 1000+*
