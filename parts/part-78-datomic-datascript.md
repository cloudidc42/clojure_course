# Part 78: Datomic และ DataScript
## ขั้นตอนที่ 2311-2340: Immutable Database, Datalog Queries, Time Travel, Pull API

---

## บทนำ

Datomic ปฏิวัติแนวคิด database:
- **Immutable facts** - ข้อมูลไม่เปลี่ยน เพิ่มเท่านั้น
- **Time travel** - query ข้อมูล ณ เวลาใดก็ได้
- **Datalog** - declarative query language
- **Pull API** - fetch subgraphs
- **DataScript** - in-memory / ClojureScript version

---

## ขั้นตอนที่ 2311: Datomic Schema

```clojure
(ns myapp.datomic.schema
  (:require [datomic.client.api :as d]))

;; Schema is also data in Datomic
(def product-schema
  [{:db/ident       :product/id
    :db/valueType   :db.type/uuid
    :db/cardinality :db.cardinality/one
    :db/unique      :db.unique/identity
    :db/doc         "Product UUID"}
   
   {:db/ident       :product/name
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/doc         "Product name"}
   
   {:db/ident       :product/price
    :db/valueType   :db.type/bigdec
    :db/cardinality :db.cardinality/one
    :db/doc         "Product price in USD"}
   
   {:db/ident       :product/categories
    :db/valueType   :db.type/ref
    :db/cardinality :db.cardinality/many
    :db/doc         "Categories this product belongs to"}
   
   {:db/ident       :category/name
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/unique      :db.unique/identity}
   
   {:db/ident       :order/id
    :db/valueType   :db.type/uuid
    :db/cardinality :db.cardinality/one
    :db/unique      :db.unique/identity}
   
   {:db/ident       :order/status
    :db/valueType   :db.type/keyword
    :db/cardinality :db.cardinality/one}
   
   {:db/ident       :order/items
    :db/valueType   :db.type/ref
    :db/cardinality :db.cardinality/many
    :db/isComponent true}  ; items are part of order
   
   {:db/ident       :order-item/product
    :db/valueType   :db.type/ref
    :db/cardinality :db.cardinality/one}
   
   {:db/ident       :order-item/quantity
    :db/valueType   :db.type/long
    :db/cardinality :db.cardinality/one}])

;; Install schema
(defn install-schema! [conn]
  (d/transact conn {:tx-data product-schema}))
```

---

## ขั้นตอนที่ 2312: Transactions

```clojure
;; Datomic transactions: add/retract facts

;; Create product
(defn create-product! [conn product]
  (let [temp-id (str "new-product-" (:id product))]
    (d/transact conn
      {:tx-data
       [{:db/id           temp-id
         :product/id      (:id product)
         :product/name    (:name product)
         :product/price   (bigdec (:price product))}
        
        ;; Transaction metadata
        {:db/id              "datomic.tx"
          :created-by         (:user-id product)
          :transaction/source "product-service"}]})))

;; Update (retract old, assert new)
(defn update-product-price! [conn product-id new-price]
  (let [db      (d/db conn)
        product (d/pull db [:db/id :product/price]
                         [:product/id product-id])]
    (d/transact conn
      {:tx-data
       [[:db/retract (:db/id product) :product/price (:product/price product)]
        [:db/add     (:db/id product) :product/price (bigdec new-price)]]})))

;; Add category to product
(defn add-product-category! [conn product-id category-name]
  (d/transact conn
    {:tx-data
     [{:db/id           [:product/id product-id]
       :product/categories {:category/name category-name}}]}))
```

---

## ขั้นตอนที่ 2313: Datalog Queries

```clojure
;; Datalog: pattern matching on facts

;; Find all products
(defn get-all-products [db]
  (d/q '[:find (pull ?p [:product/id :product/name :product/price])
         :where [?p :product/id _]]
       db))

;; Find by category
(defn get-products-by-category [db category-name]
  (d/q '[:find (pull ?p [:product/id :product/name :product/price])
         :in $ ?cat-name
         :where
         [?c :category/name ?cat-name]
         [?p :product/categories ?c]]
       db category-name))

;; Complex query: products with price range and category
(defn search-products [db category min-price max-price]
  (d/q '[:find (pull ?p [:product/id :product/name :product/price
                          {:product/categories [:category/name]}])
         :in $ ?category ?min ?max
         :where
         [?p :product/price ?price]
         [(>= ?price ?min)]
         [(<= ?price ?max)]
         [?p :product/categories ?c]
         [?c :category/name ?category]]
       db category min-price max-price))

;; Aggregations
(defn category-price-stats [db]
  (d/q '[:find ?category (min ?price) (max ?price) (avg ?price) (count ?p)
         :where
         [?p :product/price ?price]
         [?p :product/categories ?c]
         [?c :category/name ?category]]
       db))

;; Rules for reusable query logic
(def category-rules
  '[[(product-in-category? ?product ?cat-name)
     [?product :product/categories ?c]
     [?c :category/name ?cat-name]]
    
    [(expensive-product? ?product)
     [?product :product/price ?price]
     [(> ?price 100)]]])

(defn get-expensive-electronics [db]
  (d/q '[:find ?name ?price
         :in $ %
         :where
         [?p :product/name ?name]
         [?p :product/price ?price]
         (product-in-category? ?p "electronics")
         (expensive-product? ?p)]
       db category-rules))
```

---

## ขั้นตอนที่ 2314: Time Travel Queries

```clojure
;; Time travel: query at any point in time!

;; Get product as it was at a specific time
(defn get-product-as-of [conn product-id as-of-date]
  (let [db (d/as-of (d/db conn) as-of-date)]
    (d/pull db [:product/name :product/price]
             [:product/id product-id])))

;; Get history of a product
(defn get-product-history [conn product-id]
  (let [db (d/history (d/db conn))]
    (d/q '[:find ?tx ?attr ?val ?added
           :in $ ?product-id
           :where
           [?p :product/id ?product-id]
           [?p ?a ?val ?tx ?added]
           [?a :db/ident ?attr]]
         db product-id)))

;; Find what changed between two points
(defn get-changes-between [conn from-t to-t]
  (let [db-before (d/as-of (d/db conn) from-t)
        db-after  (d/as-of (d/db conn) to-t)]
    ;; Compare datoms between two database values
    (d/q '[:find ?e ?attr ?val
           :where
           [?e ?a ?val ?tx]
           [?tx :db/txInstant ?t]
           [(> ?t ?from-t)]
           [(<= ?t ?to-t)]
           [?a :db/ident ?attr]]
         (d/history (d/db conn)) from-t to-t)))

;; Audit trail: who changed what when
(defn audit-trail [conn entity-id]
  (d/q '[:find ?t ?user ?attr ?val ?added
         :in $ ?entity
         :where
         [?entity ?a ?val ?tx ?added]
         [?a :db/ident ?attr]
         [?tx :db/txInstant ?t]
         [?tx :created-by ?user]]
       (d/history (d/db conn)) entity-id))
```

---

## ขั้นตอนที่ 2315: DataScript (In-Memory / ClojureScript)

```clojure
;; DataScript: same API as Datomic but in-memory
;; Great for client-side state management
(ns myapp.client.state
  (:require [datascript.core :as ds]))

(def schema
  {:product/id      {:db/unique :db.unique/identity}
   :product/name    {}
   :cart/items      {:db/cardinality :db.cardinality/many
                     :db/valueType   :db.type/ref}
   :cart-item/product {:db/valueType :db.type/ref}
   :cart-item/qty     {}})

(defonce conn (ds/create-conn schema))

;; Add to cart
(defn add-to-cart! [product-id qty]
  (ds/transact! conn
    [{:db/id          -1  ; temp id
      :cart-item/product [:product/id product-id]
      :cart-item/qty     qty}
     {:db/id      1  ; cart entity
      :cart/items -1}]))

;; Query cart total
(defn cart-total []
  (ds/q '[:find (sum ?total)
          :with ?item
          :where
          [1 :cart/items ?item]
          [?item :cart-item/product ?p]
          [?p :product/price ?price]
          [?item :cart-item/qty ?qty]
          [(* ?price ?qty) ?total]]
        @conn))
```

---

*Part 78 จาก 100+ | ขั้นตอน 2311-2340 จาก 1000+*
