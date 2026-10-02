# Part 17: Domain-Driven Design (DDD)
## ขั้นตอนที่ 481-510: Entities, Value Objects, Aggregates, Domain Events

---

## บทนำ

Domain-Driven Design (DDD) คือแนวคิดในการออกแบบซอฟต์แวร์ที่:
- โค้ดสะท้อน **business domain** อย่างชัดเจน
- ทีมพัฒนาและผู้เชี่ยวชาญ domain ใช้ภาษาเดียวกัน (**Ubiquitous Language**)
- แบ่ง domain ออกเป็น **Bounded Contexts**

Clojure เหมาะกับ DDD เพราะ data-centric approach

---

## ขั้นตอนที่ 481: Ubiquitous Language

```
e-Commerce Domain Language:
============================
Order     = คำสั่งซื้อ
OrderItem = รายการสินค้าในคำสั่งซื้อ
Customer  = ลูกค้า
Product   = สินค้า
Cart      = ตะกร้าสินค้า
Payment   = การชำระเงิน
Shipment  = การจัดส่ง
Invoice   = ใบแจ้งหนี้

Domain Events:
==============
OrderPlaced        = วางคำสั่งซื้อแล้ว
PaymentReceived    = ได้รับการชำระเงิน
OrderShipped       = จัดส่งแล้ว
OrderDelivered     = ส่งถึงมือแล้ว
OrderCancelled     = ยกเลิกแล้ว
ProductOutOfStock  = สินค้าหมด
```

---

## ขั้นตอนที่ 482: Value Objects

```clojure
;; Value Objects: ไม่มี identity, เปรียบเทียบด้วย value

;; Email
(defrecord Email [value]
  Object
  (toString [_] value))

(defn make-email [s]
  (if (re-matches #".+@.+\..+" s)
    (->Email s)
    (throw (ex-info "Invalid email" {:value s}))))

;; Money
(defrecord Money [amount currency]
  Object
  (toString [_] (str amount " " (name currency))))

(defn make-money [amount currency]
  (when (neg? amount)
    (throw (ex-info "Amount cannot be negative" {:amount amount})))
  (->Money (bigdec amount) currency))

(defn add-money [m1 m2]
  (when (not= (:currency m1) (:currency m2))
    (throw (ex-info "Currency mismatch" {:m1 m1 :m2 m2})))
  (->Money (+ (:amount m1) (:amount m2)) (:currency m1)))

;; Address
(defrecord Address [street city state zip country])

(defn make-address [{:keys [street city state zip country]}]
  (let [errors (cond-> []
                 (blank? street)  (conj "Street is required")
                 (blank? city)    (conj "City is required")
                 (blank? zip)     (conj "ZIP is required")
                 (blank? country) (conj "Country is required"))]
    (if (seq errors)
      (throw (ex-info "Invalid address" {:errors errors}))
      (->Address street city state zip country))))
```

---

## ขั้นตอนที่ 483: Entities

```clojure
;; Entities: มี identity, เปรียบเทียบด้วย ID

;; Customer Entity
(defrecord Customer [id name email address created-at])

(defn make-customer [{:keys [name email address]}]
  {:id         (java.util.UUID/randomUUID)
   :name       name
   :email      (make-email email)
   :address    (make-address address)
   :created-at (java.time.Instant/now)})

(defn same-customer? [c1 c2]
  (= (:id c1) (:id c2)))

;; Product Entity
(defn make-product [{:keys [name price stock category]}]
  {:id       (java.util.UUID/randomUUID)
   :name     name
   :price    (make-money price :THB)
   :stock    stock
   :category (keyword category)
   :active   true})

(defn in-stock? [product qty]
  (>= (:stock product) qty))

(defn reserve-stock [product qty]
  (if (in-stock? product qty)
    (update product :stock - qty)
    (throw (ex-info "Insufficient stock"
                     {:product-id (:id product)
                      :requested  qty
                      :available  (:stock product)}))))
```

---

## ขั้นตอนที่ 484: Aggregates

```clojure
;; Aggregate Root: group ของ entities ที่มี consistency boundary

;; Order Aggregate
(defn make-order [customer-id]
  {:id          (java.util.UUID/randomUUID)
   :customer-id customer-id
   :status      :pending
   :items       []
   :total       (make-money 0 :THB)
   :created-at  (java.time.Instant/now)
   :events      []})   ; Domain events

(defn add-item-to-order [order product qty]
  (if (not= :pending (:status order))
    (throw (ex-info "Cannot modify order in status"
                     {:status (:status order)}))
    (let [item-price (make-money (* (-> product :price :amount) qty) :THB)
          item       {:product-id (:id product)
                      :name       (:name product)
                      :qty        qty
                      :unit-price (:price product)
                      :total      item-price}]
      (-> order
          (update :items conj item)
          (update :total add-money item-price)
          (update :events conj {:type      :item-added
                                  :product-id (:id product)
                                  :qty        qty
                                  :at         (java.time.Instant/now)})))))

(defn confirm-order [order]
  (if (empty? (:items order))
    (throw (ex-info "Cannot confirm empty order" {}))
    (-> order
        (assoc :status :confirmed)
        (update :events conj {:type :order-confirmed
                               :at   (java.time.Instant/now)}))))

(defn cancel-order [order reason]
  (if (#{:shipped :delivered} (:status order))
    (throw (ex-info "Cannot cancel order in status"
                     {:status (:status order)}))
    (-> order
        (assoc :status :cancelled :cancel-reason reason)
        (update :events conj {:type   :order-cancelled
                               :reason reason
                               :at     (java.time.Instant/now)}))))
```

---

## ขั้นตอนที่ 485: Domain Events

```clojure
(ns ecommerce.domain.events)

;; Domain events: สิ่งที่เกิดขึ้นใน domain
(defn order-placed [order]
  {:event/type      :order/placed
   :event/id        (java.util.UUID/randomUUID)
   :event/timestamp (java.time.Instant/now)
   :event/version   1
   :order/id        (:id order)
   :customer/id     (:customer-id order)
   :order/total     (:total order)
   :order/items     (:items order)})

(defn payment-received [order payment-ref amount]
  {:event/type      :payment/received
   :event/id        (java.util.UUID/randomUUID)
   :event/timestamp (java.time.Instant/now)
   :event/version   1
   :order/id        (:id order)
   :payment/ref     payment-ref
   :payment/amount  amount})

(defn order-shipped [order tracking-number]
  {:event/type       :order/shipped
   :event/id         (java.util.UUID/randomUUID)
   :event/timestamp  (java.time.Instant/now)
   :event/version    1
   :order/id         (:id order)
   :shipment/tracking tracking-number})

;; Event Store (simple append-only log)
(def event-store (atom []))

(defn persist-event! [event]
  (swap! event-store conj event)
  event)

(defn events-for-order [order-id]
  (filter #(= order-id (:order/id %)) @event-store))
```

---

## ขั้นตอนที่ 486: Repository Pattern (DDD Style)

```clojure
(ns ecommerce.infrastructure.repositories.order
  (:require [next.jdbc :as jdbc]
            [ecommerce.domain.order :as order]))

;; Repository interface (protocol)
(defprotocol OrderRepository
  (save! [this order])
  (find-by-id [this id])
  (find-by-customer [this customer-id])
  (find-by-status [this status page size]))

;; PostgreSQL implementation
(defrecord PostgresOrderRepository [ds]
  OrderRepository
  
  (save! [_ order]
    ;; Upsert (insert or update)
    (jdbc/with-transaction [tx ds]
      ;; Save order
      (jdbc/execute-one! tx
        ["INSERT INTO orders (id, customer_id, status, total_amount, total_currency)
          VALUES (?, ?, ?, ?, ?)
          ON CONFLICT (id) DO UPDATE
          SET status = EXCLUDED.status,
              total_amount = EXCLUDED.total_amount"
         (str (:id order))
         (str (:customer-id order))
         (name (:status order))
         (-> order :total :amount)
         (name (-> order :total :currency))])
      
      ;; Save items
      (doseq [item (:items order)]
        (jdbc/execute-one! tx
          ["INSERT INTO order_items (order_id, product_id, qty, unit_price)
            VALUES (?, ?, ?, ?)
            ON CONFLICT (order_id, product_id) DO UPDATE
            SET qty = EXCLUDED.qty"
           (str (:id order))
           (str (:product-id item))
           (:qty item)
           (-> item :unit-price :amount)]))
      
      order))
  
  (find-by-id [_ id]
    (when-let [row (jdbc/execute-one! ds
                     ["SELECT * FROM orders WHERE id = ?" (str id)])]
      (order/from-db-row row
        (jdbc/execute! ds
          ["SELECT * FROM order_items WHERE order_id = ?" (str id)])))))
```

---

## ขั้นตอนที่ 487: Application Service Layer

```clojure
(ns ecommerce.application.order-service
  (:require [ecommerce.domain.order :as order-domain]
            [ecommerce.domain.events :as events]))

;; Application Service: orchestrates domain objects

(defn place-order!
  [{:keys [order-repo product-repo event-publisher]}
   {:keys [customer-id items]}]
  
  ;; 1. Create order aggregate
  (let [new-order (order-domain/make-order customer-id)]
    
    ;; 2. Add items (validates stock)
    (let [order (reduce
                  (fn [order {:keys [product-id qty]}]
                    (let [product (product-repo/find-by-id product-id)]
                      (when-not product
                        (throw (ex-info "Product not found" {:id product-id})))
                      (order-domain/add-item-to-order order product qty)))
                  new-order
                  items)]
      
      ;; 3. Confirm order
      (let [confirmed-order (order-domain/confirm-order order)]
        
        ;; 4. Reserve stock
        (doseq [{:keys [product-id qty]} items]
          (product-repo/reserve-stock! product-id qty))
        
        ;; 5. Persist order
        (order-repo/save! confirmed-order)
        
        ;; 6. Publish domain event
        (event-publisher/publish!
          (events/order-placed confirmed-order))
        
        confirmed-order))))

(defn cancel-order!
  [{:keys [order-repo event-publisher]} order-id reason customer-id]
  
  (let [order (or (order-repo/find-by-id order-id)
                  (throw (ex-info "Order not found" {:id order-id})))]
    
    ;; Authorization: only customer can cancel their order
    (when (not= customer-id (:customer-id order))
      (throw (ex-info "Unauthorized" {:status 403})))
    
    (let [cancelled (order-domain/cancel-order order reason)]
      (order-repo/save! cancelled)
      (event-publisher/publish!
        {:event/type :order/cancelled
         :order/id   order-id
         :reason     reason})
      cancelled)))
```

---

## ขั้นตอนที่ 488: Bounded Contexts

```
E-Commerce Bounded Contexts:
==============================

┌─────────────────────┐   ┌─────────────────────┐
│   Order Context     │   │  Inventory Context  │
│                     │   │                     │
│  Order              │   │  Product            │
│  OrderItem          │   │  Stock              │
│  Customer (limited) │   │  Warehouse          │
└─────────────────────┘   └─────────────────────┘
          │ Domain Events          │
          └────────────────────────┘

┌─────────────────────┐   ┌─────────────────────┐
│  Payment Context    │   │  Shipping Context   │
│                     │   │                     │
│  Payment            │   │  Shipment           │
│  Transaction        │   │  Carrier            │
│  Invoice            │   │  Tracking           │
└─────────────────────┘   └─────────────────────┘

Context Map:
============
- Order → Inventory: "Reserve stock" (synchronous)
- Order → Payment:   "Process payment" (synchronous)  
- Order → Shipping:  "Create shipment" (via event)
- Payment → Order:   "Payment confirmed" (via event)
```

---

## Project Exercise: Complete Order Flow

```clojure
(ns ecommerce.integration-test
  (:require [clojure.test :refer :all]
            [ecommerce.application.order-service :as svc]))

(deftest test-complete-order-flow
  ;; Setup in-memory repos
  (let [products  (atom {1 {:id 1 :name "Widget" :price {:amount 100M :currency :THB} :stock 10}})
        orders    (atom {})
        events    (atom [])
        
        product-repo {:find-by-id   (fn [id] (get @products id))
                      :reserve-stock! (fn [id qty]
                                        (swap! products update-in [id :stock] - qty))}
        
        order-repo   {:save!        (fn [order] (swap! orders assoc (:id order) order))
                      :find-by-id   (fn [id] (get @orders id))}
        
        publisher    {:publish! (fn [event] (swap! events conj event))}
        
        deps {:order-repo order-repo
               :product-repo product-repo
               :event-publisher publisher}]
    
    ;; Place order
    (let [order (svc/place-order! deps
                  {:customer-id 42
                   :items [{:product-id 1 :qty 2}]})]
      
      (is (= :confirmed (:status order)))
      (is (= 1 (count (:items order))))
      
      ;; Stock was reserved
      (is (= 8 (get-in @products [1 :stock])))
      
      ;; Event was published
      (is (= 1 (count (filter #(= :order/placed (:event/type %)) @events))))
      
      ;; Cancel order
      (svc/cancel-order! deps (:id order) "Changed mind" 42)
      (is (= :cancelled (:status (get @orders (:id order))))))))
```

---

*Part 17 จาก 100+ | ขั้นตอน 481-510 จาก 1000+*
