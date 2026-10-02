# Part 90: Real-World Project Architecture
## ขั้นตอนที่ 2671-2700: E-Commerce Platform ครบสมบูรณ์

---

## บทนำ

สร้าง E-Commerce Platform ครบวงจรด้วย Clojure:
- **Clean Architecture** - domain, application, infrastructure, interface
- **Module boundaries** - clear dependency direction
- **Configuration management** - environment-based
- **System component** - Integrant lifecycle
- **Production-ready** - logging, tracing, metrics

---

## ขั้นตอนที่ 2671: Project Structure

```
ecommerce/
├── deps.edn
├── dev/
│   ├── user.clj          ; REPL utilities
│   └── dev_config.edn    ; dev configuration
├── resources/
│   ├── config.edn        ; default configuration
│   └── migrations/       ; SQL migrations
├── src/
│   ├── ecommerce/
│   │   ├── domain/       ; Pure business logic
│   │   │   ├── order.clj
│   │   │   ├── product.clj
│   │   │   ├── customer.clj
│   │   │   └── pricing.clj
│   │   ├── application/  ; Use cases / application services
│   │   │   ├── place_order.clj
│   │   │   ├── process_payment.clj
│   │   │   └── ship_order.clj
│   │   ├── infrastructure/ ; External systems
│   │   │   ├── database.clj
│   │   │   ├── payment_gateway.clj
│   │   │   ├── email_service.clj
│   │   │   └── event_bus.clj
│   │   ├── interface/    ; Adapters (HTTP, CLI, queues)
│   │   │   ├── http/
│   │   │   │   ├── routes.clj
│   │   │   │   ├── middleware.clj
│   │   │   │   └── handlers/
│   │   │   └── worker/
│   │   │       └── consumers.clj
│   │   └── system.clj    ; Integrant system config
└── test/
    ├── unit/
    ├── integration/
    └── e2e/
```

---

## ขั้นตอนที่ 2672: Domain Layer (Pure Functions)

```clojure
(ns ecommerce.domain.order)

;; Value objects
(defrecord OrderId [value])
(defrecord CustomerId [value])
(defrecord Money [amount currency])

(defn money [amount currency]
  (assert (>= amount 0) "Amount must be non-negative")
  (assert (keyword? currency) "Currency must be keyword")
  (->Money (bigdec amount) currency))

(defn add-money [m1 m2]
  (assert (= (:currency m1) (:currency m2)) "Cannot add different currencies")
  (->Money (+ (:amount m1) (:amount m2)) (:currency m1)))

;; Order aggregate
(def valid-transitions
  {:pending    #{:confirmed :cancelled}
   :confirmed  #{:paid :cancelled}
   :paid       #{:shipped}
   :shipped    #{:delivered}
   :delivered  #{}
   :cancelled  #{}})

(defn order? [x]
  (and (map? x)
       (contains? x :id)
       (contains? x :status)))

(defn can-transition? [order new-status]
  (contains? (get valid-transitions (:status order) #{}) new-status))

(defn transition! [order new-status]
  (if (can-transition? order new-status)
    (assoc order :status new-status)
    (throw (ex-info "Invalid state transition"
                     {:from   (:status order)
                      :to     new-status
                      :order  (:id order)}))))

;; Order factory
(defn create-order [order-id customer-id items]
  (let [subtotal (reduce #(add-money %1 (:line-total %2))
                          (money 0 :thb)
                          items)]
    {:id          order-id
     :customer-id customer-id
     :items       items
     :subtotal    subtotal
     :status      :pending
     :created-at  (java.time.Instant/now)
     :events      [{:type :order-created
                    :occurred-at (java.time.Instant/now)}]}))

;; Domain rules
(defn validate-order [order]
  (cond-> []
    (empty? (:items order))
    (conj "Order must have at least one item")
    
    (> (count (:items order)) 100)
    (conj "Order cannot have more than 100 items")
    
    (not (pos? (:amount (:subtotal order))))
    (conj "Order total must be positive")))
```

---

## ขั้นตอนที่ 2673: Application Layer (Use Cases)

```clojure
(ns ecommerce.application.place-order
  (:require [ecommerce.domain.order :as order]
            [ecommerce.domain.pricing :as pricing]))

;; Port (interface to infrastructure)
(defprotocol OrderRepository
  (save-order! [this order])
  (find-order  [this order-id])
  (list-orders [this customer-id]))

(defprotocol ProductRepository
  (find-product  [this product-id])
  (check-stock!  [this product-id quantity])
  (reserve-stock! [this product-id quantity]))

(defprotocol EventBus
  (publish! [this event]))

;; Use case: Place Order
(defn place-order!
  "Application service: orchestrates domain and infrastructure"
  [order-repo product-repo event-bus pricing-service
   {:keys [customer-id items] :as command}]
  
  ;; 1. Validate command
  (when (empty? items)
    (throw (ex-info "Order must have items" {:command command})))
  
  ;; 2. Load products and check availability
  (let [enriched-items
        (mapv (fn [{:keys [product-id quantity]}]
                (let [product (find-product product-repo product-id)]
                  (when-not product
                    (throw (ex-info "Product not found" {:product-id product-id})))
                  (check-stock! product-repo product-id quantity)
                  {:product-id product-id
                   :product    product
                   :quantity   quantity
                   :unit-price (:price product)
                   :line-total (order/money (* (:amount (:price product)) quantity)
                                             (:currency (:price product)))}))
               items)
        
        ;; 3. Apply pricing rules
        order-id  (str (java.util.UUID/randomUUID))
        raw-order (order/create-order order-id customer-id enriched-items)
        
        ;; 4. Validate domain rules
        errors    (order/validate-order raw-order)
        _         (when (seq errors)
                    (throw (ex-info "Order validation failed" {:errors errors})))
        
        ;; 5. Apply discounts
        discounts  (pricing/calculate-discounts pricing-service raw-order)
        final-order (pricing/apply-discounts raw-order discounts)]
    
    ;; 6. Reserve stock (infrastructure)
    (doseq [{:keys [product-id quantity]} items]
      (reserve-stock! product-repo product-id quantity))
    
    ;; 7. Persist order
    (save-order! order-repo final-order)
    
    ;; 8. Publish domain events
    (doseq [event (:events final-order)]
      (publish! event-bus (assoc event :aggregate-id order-id)))
    
    ;; 9. Return result
    final-order))
```

---

## ขั้นตอนที่ 2674: Infrastructure Layer

```clojure
(ns ecommerce.infrastructure.database
  (:require [next.jdbc :as jdbc]
            [ecommerce.application.place-order :as app]))

;; PostgreSQL implementation of OrderRepository
(defrecord PostgresOrderRepository [db]
  app/OrderRepository
  
  (save-order! [_ order]
    (jdbc/with-transaction [tx db]
      ;; Upsert order
      (jdbc/execute-one! tx
        ["INSERT INTO orders
            (id, customer_id, status, subtotal_amount, subtotal_currency,
             created_at, updated_at)
          VALUES (?, ?, ?, ?, ?, ?, NOW())
          ON CONFLICT (id) DO UPDATE
          SET status = EXCLUDED.status,
              updated_at = NOW()"
         (:id order)
         (str (:customer-id order))
         (name (:status order))
         (.setScale (:amount (:subtotal order)) 2 java.math.RoundingMode/HALF_UP)
         (name (:currency (:subtotal order)))
         (:created-at order)])
      
      ;; Upsert items
      (jdbc/execute! tx
        (into ["INSERT INTO order_items
                  (order_id, product_id, quantity, unit_price, line_total)
                VALUES (?, ?, ?, ?, ?)
                ON CONFLICT (order_id, product_id) DO UPDATE
                SET quantity = EXCLUDED.quantity"]
              (mapcat (fn [item]
                         [(:id order)
                          (:product-id item)
                          (:quantity item)
                          (:amount (:unit-price item))
                          (:amount (:line-total item))])
                      (:items order))))))
  
  (find-order [_ order-id]
    (when-let [row (jdbc/execute-one! db
                     ["SELECT o.*, json_agg(i.*) as items
                       FROM orders o
                       LEFT JOIN order_items i ON i.order_id = o.id
                       WHERE o.id = ?
                       GROUP BY o.id"
                      order-id])]
      (row->order row)))
  
  (list-orders [_ customer-id]
    (jdbc/execute! db
      ["SELECT id, status, subtotal_amount, created_at
        FROM orders
        WHERE customer_id = ?
        ORDER BY created_at DESC
        LIMIT 50"
       customer-id])))
```

---

## ขั้นตอนที่ 2675: System Configuration ด้วย Integrant

```clojure
(ns ecommerce.system
  (:require [integrant.core :as ig]
            [ecommerce.infrastructure.database :as db]
            [ecommerce.interface.http.routes :as routes]))

;; System configuration
(def config
  {:db/pool
   {:jdbc-url  #env "DATABASE_URL"
    :pool-size 10}
   
   :cache/redis
   {:url #env "REDIS_URL"}
   
   :infra/order-repo
   {:db (ig/ref :db/pool)}
   
   :infra/product-repo
   {:db (ig/ref :db/pool)}
   
   :infra/event-bus
   {:redis (ig/ref :cache/redis)}
   
   :app/place-order
   {:order-repo   (ig/ref :infra/order-repo)
    :product-repo (ig/ref :infra/product-repo)
    :event-bus    (ig/ref :infra/event-bus)}
   
   :http/server
   {:port    (or (some-> (System/getenv "PORT") Integer/parseInt) 3000)
    :handler (ig/ref :http/routes)}
   
   :http/routes
   {:place-order (ig/ref :app/place-order)}})

;; Component initialization
(defmethod ig/init-key :db/pool [_ {:keys [jdbc-url pool-size]}]
  (connection-pool/make-pool {:jdbc-url jdbc-url :max-pool-size pool-size}))

(defmethod ig/halt-key! :db/pool [_ pool]
  (connection-pool/close! pool))

(defmethod ig/init-key :http/server [_ {:keys [port handler]}]
  (let [server (ring-jetty/run-jetty handler {:port port :join? false})]
    (println "Server started on port" port)
    server))

(defmethod ig/halt-key! :http/server [_ server]
  (.stop server))

;; Start/stop system
(defonce system (atom nil))

(defn start! []
  (reset! system (ig/init config)))

(defn stop! []
  (when @system
    (ig/halt! @system)
    (reset! system nil)))
```

---

*Part 90 จาก 100+ | ขั้นตอน 2671-2700 จาก 1000+*
