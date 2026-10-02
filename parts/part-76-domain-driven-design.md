# Part 76: Domain-Driven Design (DDD)
## ขั้นตอนที่ 2251-2280: Aggregates, Value Objects, Domain Events, Bounded Contexts

---

## บทนำ

DDD concepts ใน Clojure:
- **Aggregate** - cluster of entities with invariants
- **Value Object** - immutable descriptors
- **Domain Event** - something that happened
- **Repository** - collection of aggregates
- **Domain Service** - logic that doesn't fit entity

---

## ขั้นตอนที่ 2251: Value Objects

```clojure
(ns myapp.domain.values)

;; Value Objects: equality by value, immutable

;; Email
(defn make-email [raw]
  (let [email (clojure.string/lower-case (clojure.string/trim raw))]
    (if (re-matches #"^[^\s@]+@[^\s@]+\.[^\s@]+$" email)
      {:type :email :value email}
      (throw (ex-info "Invalid email" {:raw raw})))))

(defn email-domain [email-vo]
  (second (clojure.string/split (:value email-vo) #"@")))

;; Money (as before, now as value object)
(defn make-money [amount currency]
  (when-not (>= amount 0)
    (throw (ex-info "Amount must be non-negative" {:amount amount})))
  {:type :money :amount (bigdec amount) :currency (keyword currency)})

(defn money-add [m1 m2]
  (when (not= (:currency m1) (:currency m2))
    (throw (ex-info "Currency mismatch" {})))
  (make-money (+ (:amount m1) (:amount m2)) (:currency m1)))

;; Address
(defn make-address [street city state postal-code country]
  {:type        :address
   :street      street
   :city        city
   :state       state
   :postal-code postal-code
   :country     country})

;; Quantity
(defn make-quantity [value unit]
  (when-not (pos? value)
    (throw (ex-info "Quantity must be positive" {:value value})))
  {:type :quantity :value value :unit unit})

;; DateRange
(defn make-date-range [start end]
  (when (.isAfter start end)
    (throw (ex-info "Start must be before end" {:start start :end end})))
  {:type :date-range :start start :end end})

(defn date-range-contains? [range date]
  (let [{:keys [start end]} range]
    (and (not (.isAfter start date))
         (not (.isBefore end date)))))
```

---

## ขั้นตอนที่ 2252: Aggregates

```clojure
(ns myapp.domain.order)

;; Order Aggregate: Order + OrderItems + OrderEvents

;; Order states
(def valid-transitions
  {:pending    #{:confirmed :cancelled}
   :confirmed  #{:processing :cancelled}
   :processing #{:shipped :cancelled}
   :shipped    #{:delivered}
   :delivered  #{}
   :cancelled  #{}})

(defn can-transition? [from to]
  (contains? (get valid-transitions from #{}) to))

;; Create order aggregate
(defn create-order [customer-id items]
  (when (empty? items)
    (throw (ex-info "Order must have at least one item" {})))
  
  (let [now (java.time.Instant/now)]
    {:id          (str (java.util.UUID/randomUUID))
     :customer-id customer-id
     :status      :pending
     :items       items
     :subtotal    (reduce + (map #(* (:price %) (:quantity %)) items))
     :created-at  now
     :events      [{:type      :order-created
                    :occurred-at now
                    :payload   {:customer-id customer-id
                                :item-count  (count items)}}]}))

;; Aggregate commands (return new state + events)
(defn confirm-order [order]
  (when-not (can-transition? (:status order) :confirmed)
    (throw (ex-info "Cannot confirm order in current state"
                     {:status (:status order)})))
  
  (let [event {:type        :order-confirmed
                :occurred-at (java.time.Instant/now)}]
    (-> order
        (assoc :status :confirmed)
        (update :events conj event))))

(defn add-item [order item]
  (when (not= :pending (:status order))
    (throw (ex-info "Cannot modify confirmed order" {})))
  
  (let [event {:type     :item-added
                :payload  item}]
    (-> order
        (update :items conj item)
        (update :subtotal + (* (:price item) (:quantity item)))
        (update :events conj event))))

(defn apply-discount [order discount-pct]
  (when (> discount-pct 0.5)
    (throw (ex-info "Discount cannot exceed 50%" {:discount discount-pct})))
  
  (let [discount (* (:subtotal order) discount-pct)]
    (-> order
        (assoc :discount discount-pct)
        (assoc :total (- (:subtotal order) discount)))))

(defn cancel-order [order reason]
  (when-not (can-transition? (:status order) :cancelled)
    (throw (ex-info "Cannot cancel order" {:status (:status order)})))
  
  (-> order
      (assoc :status :cancelled :cancelled-reason reason)
      (update :events conj {:type    :order-cancelled
                             :payload {:reason reason}})))
```

---

## ขั้นตอนที่ 2253: Repository Pattern

```clojure
(ns myapp.domain.repository)

;; Repository: abstraction over persistence
(defprotocol OrderRepository
  (find-by-id [repo order-id])
  (find-by-customer [repo customer-id pagination])
  (save! [repo order])
  (delete! [repo order-id]))

;; PostgreSQL implementation
(defrecord PostgresOrderRepository [db]
  OrderRepository
  
  (find-by-id [_ order-id]
    (when-let [row (jdbc/execute-one! db
                     ["SELECT * FROM orders WHERE id = ?" order-id])]
      ;; Reconstruct aggregate from persistence
      (-> row
          (update :orders/status keyword)
          (assoc :orders/items (get-order-items db order-id)))))
  
  (find-by-customer [_ customer-id {:keys [page size]}]
    (->> (jdbc/execute! db
           ["SELECT * FROM orders WHERE customer_id = ?
             ORDER BY created_at DESC LIMIT ? OFFSET ?"
            customer-id size (* (dec page) size)])
         (map #(update % :orders/status keyword))))
  
  (save! [_ order]
    (jdbc/with-transaction [tx db]
      ;; Upsert order
      (jdbc/execute-one! tx
        ["INSERT INTO orders (id, customer_id, status, subtotal, total, discount)
          VALUES (?, ?, ?, ?, ?, ?)
          ON CONFLICT (id) DO UPDATE SET
            status = EXCLUDED.status,
            subtotal = EXCLUDED.subtotal,
            total = EXCLUDED.total,
            discount = EXCLUDED.discount"
         (:id order) (:customer-id order) (name (:status order))
         (:subtotal order) (:total order) (:discount order)])
      
      ;; Save new events
      (let [new-events (filter #(not (:persisted? %)) (:events order))]
        (doseq [event new-events]
          (jdbc/execute-one! tx
            ["INSERT INTO order_events (order_id, type, occurred_at, payload)
              VALUES (?, ?, ?, ?::jsonb)"
             (:id order) (name (:type event))
             (:occurred-at event) (json/generate-string (:payload event))])))
      
      order)))

;; In-memory for testing
(defrecord InMemoryOrderRepository [store]
  OrderRepository
  (find-by-id [_ order-id] (get @store order-id))
  (save! [_ order] (swap! store assoc (:id order) order) order)
  (delete! [_ order-id] (swap! store dissoc order-id)))

(defn in-memory-repo [] (->InMemoryOrderRepository (atom {})))
```

---

## ขั้นตอนที่ 2254: Domain Services

```clojure
(ns myapp.domain.services)

;; Domain services: business logic that spans multiple aggregates

(defn transfer-order-ownership!
  "Transfer order from one customer to another (business rule)"
  [order-repo customer-repo order-id new-customer-id]
  (let [order    (find-by-id order-repo order-id)
        customer (find-by-id customer-repo new-customer-id)]
    
    ;; Business rules
    (when-not order
      (throw (ex-info "Order not found" {:id order-id})))
    (when-not customer
      (throw (ex-info "Customer not found" {:id new-customer-id})))
    (when (not= :pending (:status order))
      (throw (ex-info "Can only transfer pending orders" {})))
    (when (not (:active? customer))
      (throw (ex-info "Cannot transfer to inactive customer" {})))
    
    (let [updated-order (assoc order :customer-id new-customer-id)]
      (save! order-repo updated-order)
      updated-order)))

;; Pricing service: complex pricing rules
(defn calculate-final-price
  [base-price quantity customer-tier coupon-code]
  (let [volume-discount (cond
                           (>= quantity 100) 0.20
                           (>= quantity 50)  0.10
                           (>= quantity 10)  0.05
                           :else             0)
        
        tier-discount   (case customer-tier
                           :gold     0.15
                           :silver   0.10
                           :bronze   0.05
                           :standard 0)
        
        coupon-discount (when coupon-code
                          (get-coupon-discount coupon-code))
        
        ;; Business rule: max total discount 40%
        total-discount  (min 0.40 (+ volume-discount tier-discount
                                      (or coupon-discount 0)))
        
        unit-price  (* base-price (- 1 total-discount))
        total-price (* unit-price quantity)]
    
    {:unit-price      unit-price
     :total-price     total-price
     :discounts       {:volume  volume-discount
                       :tier    tier-discount
                       :coupon  coupon-discount}
     :total-discount  total-discount}))
```

---

## ขั้นตอนที่ 2255: Bounded Context Integration

```clojure
(ns myapp.bounded-context)

;; Anti-corruption layer between bounded contexts
;; Prevents concepts from one context polluting another

;; Order context receives notification from Shipping context
(defn shipping-event->order-event
  "Translate shipping domain event to order domain event"
  [shipping-event]
  (case (:type shipping-event)
    :shipment-dispatched
    {:type       :order-shipped
     :order-id   (:order-id shipping-event)
     :carrier    (:carrier shipping-event)
     :tracking   (:tracking-number shipping-event)
     :eta        (:estimated-delivery shipping-event)}
    
    :shipment-delivered
    {:type     :order-delivered
     :order-id (:order-id shipping-event)
     :signed-by (:recipient shipping-event)}
    
    nil))

;; Context map shows how bounded contexts relate
(def bounded-contexts
  {:order-context    {:owns #{:orders :order-items}
                       :uses #{:customer-id :product-id}}
   :catalog-context  {:owns #{:products :categories}
                       :uses #{}}
   :customer-context {:owns #{:customers :addresses}
                       :uses #{}}
   :shipping-context {:owns #{:shipments :carriers}
                       :uses #{:order-id}}})
```

---

*Part 76 จาก 100+ | ขั้นตอน 2251-2280 จาก 1000+*
