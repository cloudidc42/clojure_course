# Part 34: Advanced Domain Modeling
## ขั้นตอนที่ 991-1020: Rich Domain Models, Value Objects, Bounded Contexts

---

## บทนำ

Domain-Driven Design (DDD) ขั้นสูง:
- **Value Objects** ที่ smart และ self-validating
- **Domain Events** ที่ rich
- **Aggregate Roots** ที่มี business invariants
- **Domain Services** สำหรับ cross-aggregate operations
- **Anti-Corruption Layer** ระหว่าง bounded contexts

---

## ขั้นตอนที่ 991: Rich Value Objects

```clojure
(ns domain.values)

;; ===== Money Value Object =====
;; Currency-aware, arithmetic operations

(defrecord Money [amount currency]
  Object
  (toString [_] (str amount " " (name currency))))

(defn money [amount currency]
  {:pre [(number? amount)
         (keyword? currency)
         (>= amount 0)]}
  (->Money (bigdec amount) currency))

(defn add-money [m1 m2]
  (when-not (= (:currency m1) (:currency m2))
    (throw (ex-info "Cannot add different currencies"
                     {:m1 m1 :m2 m2})))
  (money (+ (:amount m1) (:amount m2)) (:currency m1)))

(defn multiply-money [m factor]
  (money (* (:amount m) factor) (:currency m)))

(defn compare-money [m1 m2]
  (when-not (= (:currency m1) (:currency m2))
    (throw (ex-info "Cannot compare different currencies"
                     {:m1 m1 :m2 m2})))
  (compare (:amount m1) (:amount m2)))

;; ===== Email Value Object =====

(defrecord Email [address]
  Object
  (toString [_] address))

(defn ->email [s]
  (when-not (re-matches #".+@.+\..+" s)
    (throw (ex-info "Invalid email address" {:email s})))
  (->Email (clojure.string/lower-case s)))

;; ===== Address Value Object =====

(defrecord Address [street city province zip-code country]
  Object
  (toString [_] (str street ", " city ", " province " " zip-code ", " country)))

(defn ->address [m]
  (let [{:keys [street city province zip-code country]} m]
    (when (some empty? [street city province zip-code country])
      (throw (ex-info "Address fields cannot be empty" {:address m})))
    (map->Address m)))
```

---

## ขั้นตอนที่ 992: Domain Events

```clojure
(ns domain.events)

;; Base event
(defprotocol DomainEvent
  (aggregate-id [e])
  (occurred-at [e])
  (event-type [e]))

;; Event factory
(defn ->event [type agg-id data]
  {:event-type  type
   :aggregate-id agg-id
   :occurred-at (java.time.Instant/now)
   :id          (java.util.UUID/randomUUID)
   :data        data})

;; Order events
(defn order-placed [order-id user-id items total]
  (->event :order/placed order-id
    {:user-id user-id :items items :total total}))

(defn order-confirmed [order-id]
  (->event :order/confirmed order-id {}))

(defn order-shipped [order-id tracking-number]
  (->event :order/shipped order-id
    {:tracking-number tracking-number}))

(defn order-cancelled [order-id reason]
  (->event :order/cancelled order-id
    {:reason reason}))

;; Event subscriber (for read models)
(defmulti apply-event (fn [state event] (:event-type event)))

(defmethod apply-event :order/placed [state event]
  (let [{:keys [user-id items total]} (:data event)]
    (assoc state
      :status  :placed
      :user-id user-id
      :items   items
      :total   total)))

(defmethod apply-event :order/confirmed [state _]
  (assoc state :status :confirmed))

(defmethod apply-event :order/shipped [state event]
  (assoc state
    :status          :shipped
    :tracking-number (get-in event [:data :tracking-number])))
```

---

## ขั้นตอนที่ 993: Aggregate Root ที่สมบูรณ์

```clojure
(ns domain.order)

;; Order Aggregate
(defn create-order [order-id user-id]
  {:id          order-id
   :user-id     user-id
   :status      :draft
   :items       []
   :total       (money 0 :THB)
   :events      []  ; uncommitted events
   :created-at  (java.time.Instant/now)})

;; Domain operations with business rules
(defn add-item
  "Add item to order. Throws if order not in draft state."
  [order product qty]
  {:pre [(pos? qty)
         (number? (:price product))]}
  
  (when-not (= :draft (:status order))
    (throw (ex-info "Cannot add item: order not in draft"
                     {:order-id (:id order) :status (:status order)})))
  
  (let [item  {:product-id (:id product)
                :name       (:name product)
                :price      (:price product)
                :qty        qty
                :subtotal   (* (:price product) qty)}
        event {:type :item-added :product product :qty qty}]
    
    (-> order
        (update :items conj item)
        (update :total add-money (money (* (:price product) qty) :THB))
        (update :events conj event))))

(defn confirm-order [order]
  (when (empty? (:items order))
    (throw (ex-info "Cannot confirm empty order"
                     {:order-id (:id order)})))
  (when-not (= :draft (:status order))
    (throw (ex-info "Order already confirmed"
                     {:order-id (:id order) :status (:status order)})))
  
  (let [event (order-confirmed (:id order))]
    (-> order
        (assoc :status :confirmed)
        (update :events conj event))))

(defn apply-discount
  "Apply percentage discount. Max 50%."
  [order discount-pct]
  (when (> discount-pct 50)
    (throw (ex-info "Discount cannot exceed 50%"
                     {:discount discount-pct})))
  
  (let [discount-factor (- 1 (/ discount-pct 100.0))
        new-total       (multiply-money (:total order) discount-factor)
        event           {:type :discount-applied :pct discount-pct}]
    
    (-> order
        (assoc :total new-total)
        (assoc :discount-pct discount-pct)
        (update :events conj event))))

;; Extract and clear uncommitted events
(defn extract-events [aggregate]
  (let [events (:events aggregate)]
    [(assoc aggregate :events []) events]))
```

---

## ขั้นตอนที่ 994: Repository Pattern ขั้นสูง

```clojure
(ns domain.repositories)

;; Repository protocol (abstract)
(defprotocol OrderRepository
  (find-by-id       [repo id])
  (find-by-user     [repo user-id opts])
  (save!            [repo order])
  (find-by-status   [repo status opts])
  (count-by-user    [repo user-id]))

;; PostgreSQL implementation
(defrecord PostgresOrderRepository [ds]
  OrderRepository
  
  (find-by-id [_ id]
    (when-let [row (db/get-by-id ds :orders id)]
      (-> row
          (update :items read-string)
          (update :total #(money % :THB))
          (update :status keyword))))
  
  (find-by-user [_ user-id {:keys [page limit status]}]
    (let [q {:select [:*]
              :from   :orders
              :where  (cond-> [:= :user_id user-id]
                        status (conj [:= :status (name status)]))
              :limit  (or limit 20)
              :offset (* (dec (or page 1)) (or limit 20))}]
      (->> (db/query ds (hsql/format q))
           (map #(-> %
                     (update :items read-string)
                     (update :status keyword))))))
  
  (save! [_ order]
    (let [row {:id      (:id order)
                :user_id (:user-id order)
                :status  (name (:status order))
                :items   (pr-str (:items order))
                :total   (str (get-in order [:total :amount]))
                :created_at (:created-at order)}]
      (db/upsert! ds :orders row)
      ;; Publish domain events
      (doseq [event (:events order)]
        (event-publisher/publish! event))
      order)))

;; In-memory implementation for tests
(defrecord InMemoryOrderRepository [store]
  OrderRepository
  
  (find-by-id [_ id]
    (get @store id))
  
  (save! [_ order]
    (swap! store assoc (:id order) order)
    order)
  
  (find-by-user [_ user-id {:keys [status]}]
    (->> (vals @store)
         (filter #(= user-id (:user-id %)))
         (filter #(or (nil? status) (= status (:status %)))))))
```

---

## ขั้นตอนที่ 995: Domain Services

```clojure
(ns domain.services)

;; Price Calculator Service (cross-entity logic)
(defprotocol PricingService
  (calculate-total    [this order])
  (apply-promotions   [this order user])
  (calculate-shipping [this order address]))

(defrecord StandardPricingService [promotion-repo shipping-calculator]
  PricingService
  
  (calculate-total [_ order]
    (->> (:items order)
         (map #(* (:price %) (:qty %)))
         (reduce +)))
  
  (apply-promotions [_ order user]
    (let [promos (promotion-repo/find-applicable promotion-repo
                                                  order user)]
      (reduce (fn [order promo]
                (case (:type promo)
                  :percentage (apply-discount order (:discount promo))
                  :free-shipping (assoc order :free-shipping? true)
                  :gift (add-gift-item order (:gift-product-id promo))
                  order))
              order
              promos)))
  
  (calculate-shipping [_ order address]
    (if (:free-shipping? order)
      (money 0 :THB)
      (shipping-calculator/calculate
        {:weight  (total-weight (:items order))
         :address address
         :express (:express? order)}))))

;; Order Application Service
(defn place-order!
  "Orchestrates order placement"
  [user-id items address pricing-service order-repo inventory-repo]
  
  ;; Check inventory
  (doseq [item items]
    (when-not (inventory-repo/available? inventory-repo
                                          (:product-id item)
                                          (:qty item))
      (throw (ex-info "Item out of stock"
                       {:product-id (:product-id item)}))))
  
  ;; Create order
  (let [order-id  (java.util.UUID/randomUUID)
        user      (user-repo/find-by-id user-id)
        order     (reduce (fn [o item]
                             (add-item o (:product item) (:qty item)))
                           (create-order order-id user-id)
                           items)
        shipping  (calculate-shipping pricing-service order address)
        order     (-> order
                      (assoc :shipping shipping)
                      (assoc :address address))
        order     (apply-promotions pricing-service order user)
        [order events] (confirm-order order)]
    
    ;; Save and publish
    (order-repo/save! order-repo order)
    {:order  order :events events}))
```

---

## Project: E-commerce Domain Model

```clojure
(ns ecommerce.domain)

;; Complete domain model for e-commerce

;; ===== Catalog Bounded Context =====
(defn create-product [id name price category]
  {:type      :product
   :id        id
   :name      name
   :price     (money price :THB)
   :category  category
   :status    :active
   :created-at (java.time.Instant/now)})

;; ===== Customer Bounded Context =====
(defn create-customer [id name email]
  {:type      :customer
   :id        id
   :name      name
   :email     (->email email)
   :addresses []
   :tier      :standard})

(defn add-address [customer address]
  (update customer :addresses conj (->address address)))

;; ===== Order Bounded Context =====
;; Uses references to catalog/customer (by ID only)

(defn checkout!
  [customer-id product-ids-qtys address-id db]
  (let [customer (customer-repo/find db customer-id)
        address  (find-address customer address-id)
        items    (map (fn [[product-id qty]]
                        {:product (catalog-repo/find db product-id)
                         :qty     qty})
                      product-ids-qtys)]
    (place-order! customer-id items address pricing-service
                   (order-repo/new db)
                   (inventory-repo/new db))))
```

---

*Part 34 จาก 100+ | ขั้นตอน 991-1020 จาก 1000+*
