# Part 46: Datomic และ Datalog
## ขั้นตอนที่ 1351-1380: Immutable Database, Datalog Queries, Time Travel, Schema

---

## บทนำ

Datomic - Immutable, Time-aware Database:
- **Accumulate-only** - ข้อมูลไม่เคยถูกลบ แต่ retract ได้
- **Datalog** - declarative query language
- **Time Travel** - query ณ เวลาใดก็ได้ในอดีต
- **Pull API** - nested data fetching
- **Speculative Writes** - `with` ทดสอบก่อน commit

---

## ขั้นตอนที่ 1351: Schema และ Connection

```clojure
;; deps.edn
;; {:deps {com.datomic/peer {:mvn/version "1.0.7187"}}}
;; หรือ Datomic Free / Datomic Cloud

(ns myapp.datomic.schema
  (:require [datomic.api :as d]))

;; Schema definition
(def schema
  [;; User attributes
   {:db/ident       :user/id
    :db/valueType   :db.type/uuid
    :db/cardinality :db.cardinality/one
    :db/unique      :db.unique/identity
    :db/doc         "User unique ID"}
   
   {:db/ident       :user/email
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/unique      :db.unique/identity
    :db/doc         "User email address"}
   
   {:db/ident       :user/name
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one}
   
   {:db/ident       :user/role
    :db/valueType   :db.type/keyword
    :db/cardinality :db.cardinality/one}
   
   ;; Product attributes
   {:db/ident       :product/id
    :db/valueType   :db.type/uuid
    :db/cardinality :db.cardinality/one
    :db/unique      :db.unique/identity}
   
   {:db/ident       :product/name
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/fulltext    true}
   
   {:db/ident       :product/price
    :db/valueType   :db.type/bigdec
    :db/cardinality :db.cardinality/one}
   
   {:db/ident       :product/category
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/index       true}
   
   {:db/ident       :product/tags
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/many}  ; Multiple values!
   
   ;; Order attributes
   {:db/ident       :order/id
    :db/valueType   :db.type/uuid
    :db/cardinality :db.cardinality/one
    :db/unique      :db.unique/identity}
   
   {:db/ident       :order/user
    :db/valueType   :db.type/ref
    :db/cardinality :db.cardinality/one}
   
   {:db/ident       :order/status
    :db/valueType   :db.type/keyword
    :db/cardinality :db.cardinality/one}
   
   {:db/ident       :order/items
    :db/valueType   :db.type/ref
    :db/cardinality :db.cardinality/many
    :db/isComponent true}  ; Items are part of order
   
   {:db/ident       :order/total
    :db/valueType   :db.type/bigdec
    :db/cardinality :db.cardinality/one}])

;; Connect and setup
(defn create-db! [uri]
  (d/create-database uri)
  (let [conn (d/connect uri)]
    @(d/transact conn schema)
    conn))

;; Sample
(def conn (create-db! "datomic:free://localhost:4334/myapp"))
```

---

## ขั้นตอนที่ 1352: Transactions (Write)

```clojure
(ns myapp.datomic.write
  (:require [datomic.api :as d]))

;; Add entity
(defn create-user! [conn user]
  (let [result @(d/transact conn
                  [{:db/id         (d/tempid :db.part/user)
                    :user/id       (:id user)
                    :user/email    (:email user)
                    :user/name     (:name user)
                    :user/role     (:role user)}])]
    ;; Return the entity after transaction
    (d/entity (:db-after result)
               [:user/id (:id user)])))

;; Update entity (Datomic adds new facts, keeps old)
(defn update-user-name! [conn user-id new-name]
  @(d/transact conn
     [{:db/id     [:user/id user-id]
       :user/name new-name}]))

;; Retract a fact (not delete, just retract)
(defn remove-user-tag! [conn user-id tag]
  @(d/transact conn
     [[:db/retract [:user/id user-id] :user/tags tag]]))

;; Component entities (order with items)
(defn create-order! [conn order]
  @(d/transact conn
     [(merge
        {:db/id        (d/tempid :db.part/user)
         :order/id     (:id order)
         :order/user   [:user/id (:user-id order)]
         :order/status :order.status/pending
         :order/total  (bigdec (:total order))
         :order/items  (map (fn [item]
                               {:db/id          (d/tempid :db.part/user)
                                :order-item/product [:product/id (:product-id item)]
                                :order-item/qty     (:qty item)
                                :order-item/price   (bigdec (:price item))})
                             (:items order))})]))

;; Transaction with metadata
(defn update-order-status! [conn order-id new-status user-who-updated]
  @(d/transact conn
     [{:db/id        [:order/id order-id]
       :order/status new-status}
      
      ;; Transaction metadata
      {:db/id           "datomic.tx"
       :tx/updated-by   [:user/id user-who-updated]
       :tx/reason       "status-change"}]))
```

---

## ขั้นตอนที่ 1353: Datalog Queries

```clojure
(ns myapp.datomic.query
  (:require [datomic.api :as d]))

;; Simple query
(defn find-user-by-email [db email]
  (d/q '[:find (pull ?u [:user/id :user/name :user/email :user/role])
         :in $ ?email
         :where [?u :user/email ?email]]
       db email))

;; Find with conditions
(defn find-active-products [db category]
  (d/q '[:find (pull ?p [:product/id :product/name :product/price :product/category])
         :in $ ?cat
         :where
         [?p :product/category ?cat]
         [?p :product/status :product.status/active]
         [(> ?price 0)]
         [?p :product/price ?price]]
       db category))

;; Find with aggregation
(defn order-stats-by-user [db]
  (d/q '[:find ?user-email (count ?order) (sum ?total)
         :keys user-email order-count total-revenue
         :where
         [?order :order/user ?user]
         [?user  :user/email ?user-email]
         [?order :order/total ?total]
         [?order :order/status :order.status/completed]]
       db))

;; Recursive query
(defn find-category-tree [db root-category]
  (d/q '[:find ?cat ?parent
         :in $ % ?root
         :where (ancestor? ?root ?cat ?parent)]
       db
       '[[(ancestor? ?root ?cat ?parent)
          [?cat :category/parent ?root]
          [(ground ?root) ?parent]]
         [(ancestor? ?root ?cat ?parent)
          [?mid :category/parent ?root]
          (ancestor? ?mid ?cat ?parent)]]
       root-category))

;; Pull syntax for nested data
(defn get-order-full [db order-id]
  (d/pull db
    '[:order/id
      :order/status
      :order/total
      {:order/user [:user/id :user/name :user/email]}
      {:order/items
       [:order-item/qty :order-item/price
        {:order-item/product [:product/id :product/name]}]}]
    [:order/id order-id]))
```

---

## ขั้นตอนที่ 1354: Time Travel Queries

```clojure
(ns myapp.datomic.time-travel
  (:require [datomic.api :as d]))

;; Query at a specific point in time
(defn get-order-at [conn order-id as-of-time]
  (let [db-at (d/as-of (d/db conn) as-of-time)]
    (d/pull db-at
      '[:order/status :order/total]
      [:order/id order-id])))

;; History of a specific entity
(defn get-order-history [conn order-id]
  (let [db   (d/db conn)
        hist (d/history db)]
    (d/q '[:find ?tx ?attr ?val ?added ?tx-time
           :in $ ?order-id
           :where
           [?order :order/id ?order-id]
           [?order ?attr-id ?val ?tx ?added]
           [?attr-id :db/ident ?attr]
           [(#{:order/status :order/total} ?attr)]
           [?tx :db/txInstant ?tx-time]]
         hist order-id)))

;; Who changed what and when
(defn audit-log [conn entity-id since]
  (let [db   (d/db conn)
        hist (d/history db)]
    (->> (d/q '[:find ?tx-time ?attr ?old-val ?new-val ?changed-by
                :in $ ?e ?since
                :where
                [?e ?attr-id ?new-val ?tx true]
                [?e ?attr-id ?old-val ?tx false]
                [?attr-id :db/ident ?attr]
                [?tx :db/txInstant ?tx-time]
                [?tx :tx/updated-by ?user]
                [?user :user/email ?changed-by]
                [(> ?tx-time ?since)]]
             hist entity-id since)
         (sort-by first))))

;; "What if" with speculative writes
(defn simulate-order-discount [conn order-id discount-pct]
  (let [db     (d/db conn)
        order  (d/pull db '[:order/total] [:order/id order-id])
        new-total (* (:order/total order) (- 1 (/ discount-pct 100.0)))
        
        ;; Speculative transaction - doesn't actually commit
        spec-db (d/with db [{:db/id      [:order/id order-id]
                              :order/total new-total}])]
    
    ;; Query the speculative state
    {:original-total (:order/total order)
     :discounted-total new-total
     :savings          (- (:order/total order) new-total)
     :simulated-order  (d/pull (:db-after spec-db)
                                '[:order/total :order/status]
                                [:order/id order-id])}))
```

---

## ขั้นตอนที่ 1355: Datomic Rules

```clojure
;; Reusable query rules
(def rules
  '[;; Active product rule
    [(active-product? ?p)
     [?p :product/status :product.status/active]
     [?p :product/stock ?stock]
     [(> ?stock 0)]]
    
    ;; Premium user rule
    [(premium-user? ?u)
     [?u :user/tier :user.tier/premium]]
    [(premium-user? ?u)
     [?u :user/order-count ?count]
     [(>= ?count 10)]]
    
    ;; Recent order rule
    [(recent-order? ?o ?days)
     [?o :order/created-at ?date]
     [(java.time.Instant/now) ?now]
     [(- ?now (* ?days 86400000)) ?threshold]
     [(> ?date ?threshold)]]])

;; Use rules in query
(defn find-premium-user-recent-orders [db days]
  (d/q '[:find (pull ?o [:order/id :order/total :order/status])
         :in $ % ?days
         :where
         [?o :order/user ?u]
         (premium-user? ?u)
         (recent-order? ?o ?days)]
       db rules days))
```

---

## Project: Immutable Blog Platform

```clojure
(ns blog.datomic
  (:require [datomic.api :as d]))

;; Blog schema
(def blog-schema
  [{:db/ident :post/id        :db/valueType :db.type/uuid    :db/cardinality :db.cardinality/one :db/unique :db.unique/identity}
   {:db/ident :post/title     :db/valueType :db.type/string  :db/cardinality :db.cardinality/one :db/fulltext true}
   {:db/ident :post/content   :db/valueType :db.type/string  :db/cardinality :db.cardinality/one}
   {:db/ident :post/author    :db/valueType :db.type/ref     :db/cardinality :db.cardinality/one}
   {:db/ident :post/tags      :db/valueType :db.type/string  :db/cardinality :db.cardinality/many}
   {:db/ident :post/published :db/valueType :db.type/boolean :db/cardinality :db.cardinality/one}])

;; Get all versions of a post title (editorial history)
(defn post-title-history [conn post-id]
  (let [hist (d/history (d/db conn))]
    (->> (d/q '[:find ?tx-time ?title ?added
                :in $ ?post-id
                :where
                [?post :post/id ?post-id]
                [?post :post/title ?title ?tx ?added]
                [?tx :db/txInstant ?tx-time]]
             hist post-id)
         (sort-by first)
         (map (fn [[time title added]]
                {:time  time
                 :title title
                 :op    (if added :added :retracted)})))))

;; Restore a previous version
(defn restore-post-version! [conn post-id as-of-time]
  (let [db-at   (d/as-of (d/db conn) as-of-time)
        old-ver (d/pull db-at '[:post/title :post/content] [:post/id post-id])]
    @(d/transact conn [(assoc old-ver :db/id [:post/id post-id]
                                       :post/restored-from as-of-time)])))
```

---

*Part 46 จาก 100+ | ขั้นตอน 1351-1380 จาก 1000+*
