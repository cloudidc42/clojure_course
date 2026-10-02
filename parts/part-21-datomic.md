# Part 21: Datomic - Immutable Database
## ขั้นตอนที่ 601-630: Datalog Queries, Time Travel, Schema, Transactions

---

## บทนำ

**Datomic** คือ database ที่ออกแบบสำหรับ Clojure:
- **Immutable facts** - ข้อมูลไม่ถูก delete/update จริงๆ มีแค่ assertion/retraction
- **Time travel** - query ข้อมูล ณ เวลาใดก็ได้
- **Datalog** - query language ที่ declarative และ powerful
- **Peer model** - read ใน app process เอง

Datomic Free tier ใช้ได้ฟรี, Datomic Pro สำหรับ production

---

## ขั้นตอนที่ 601: Datomic Concepts

```
Traditional DB vs Datomic:
==========================

Traditional:         Datomic:
┌──────────────┐     Facts (Datoms):
│ users table  │     [entity attribute value tx added?]
│ id name age  │     [100  :user/name  "สมชาย"  1000  true]
│ 1  สมชาย  25 │     [100  :user/age   25       1000  true]
│ 2  สมหญิง 30 │     [100  :user/age   26       2000  true]  ← update!
└──────────────┘     [100  :user/age   25       1000  false] ← retracted
                     
                     Query ณ tx 1000: age = 25
                     Query ณ tx 2000: age = 26
                     Time travel!
```

---

## ขั้นตอนที่ 602: Schema Definition

```clojure
(require '[datomic.api :as d])

;; Connect to database
(def conn (d/connect "datomic:free://localhost:4334/myapp"))

;; หรือ in-memory
(d/create-database "datomic:mem://myapp")
(def conn (d/connect "datomic:mem://myapp"))

;; Schema = Datomic entities ที่อธิบาย attributes
(def schema
  [{:db/ident       :user/id
    :db/valueType   :db.type/uuid
    :db/cardinality :db.cardinality/one
    :db/unique      :db.unique/identity
    :db/doc         "User's unique ID"}
   
   {:db/ident       :user/name
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/doc         "User's full name"}
   
   {:db/ident       :user/email
    :db/valueType   :db.type/string
    :db/cardinality :db.cardinality/one
    :db/unique      :db.unique/value
    :db/doc         "User's email (unique)"}
   
   {:db/ident       :user/age
    :db/valueType   :db.type/long
    :db/cardinality :db.cardinality/one}
   
   ;; Many cardinality: user can have multiple roles
   {:db/ident       :user/roles
    :db/valueType   :db.type/keyword
    :db/cardinality :db.cardinality/many}
   
   ;; Reference to another entity
   {:db/ident       :order/user
    :db/valueType   :db.type/ref
    :db/cardinality :db.cardinality/one}
   
   {:db/ident       :order/status
    :db/valueType   :db.type/keyword
    :db/cardinality :db.cardinality/one}
   
   {:db/ident       :order/total
    :db/valueType   :db.type/bigdec
    :db/cardinality :db.cardinality/one}])

;; Apply schema
@(d/transact conn schema)
```

---

## ขั้นตอนที่ 603: Transactions (Writing Data)

```clojure
;; Transact = add facts to database

;; Add new entity
(let [result @(d/transact conn
                [{:db/id     "temp-user-1"  ; temp ID
                  :user/id   (java.util.UUID/randomUUID)
                  :user/name "สมชาย"
                  :user/email "somchai@test.com"
                  :user/age   25
                  :user/roles [:admin :user]}])]
  (println "Transaction result:" (:db-before result))
  (println "Entity ID:" (d/resolve-tempid (:db-after result)
                                           (:tempids result)
                                           "temp-user-1")))

;; Add multiple entities
@(d/transact conn
  [{:db/id       "user-1"
    :user/id     #uuid "a0be1234-0000-0000-0000-000000000001"
    :user/name   "สมชาย"
    :user/email  "a@test.com"
    :user/age    25}
   
   {:db/id       "user-2"
    :user/id     #uuid "a0be1234-0000-0000-0000-000000000002"
    :user/name   "สมหญิง"
    :user/email  "b@test.com"
    :user/age    30}
   
   {:db/id         "order-1"
    :order/user    "user-1"  ; reference!
    :order/status  :pending
    :order/total   1500.00M}])

;; Retract (soft delete)
@(d/transact conn [[:db/retractEntity [:user/email "a@test.com"]]])

;; Retract specific attribute
@(d/transact conn [[:db/retract [:user/email "b@test.com"] :user/age 30]])
```

---

## ขั้นตอนที่ 604: Datalog Queries

```clojure
;; db = snapshot of database at a point in time
(def db (d/db conn))

;; Basic query: find all users
(d/q '[:find ?e ?name ?email
       :where
       [?e :user/name ?name]
       [?e :user/email ?email]]
     db)
;; => #{[100 "สมชาย" "a@test.com"] [101 "สมหญิง" "b@test.com"]}

;; Query with filters
(d/q '[:find ?name ?age
       :where
       [?e :user/name ?name]
       [?e :user/age ?age]
       [(> ?age 20)]]
     db)

;; Query with input parameters
(defn find-user-by-email [db email]
  (d/q '[:find ?e ?name ?age
         :in $ ?email
         :where
         [?e :user/email ?email]
         [?e :user/name ?name]
         [?e :user/age ?age]]
       db email))

(find-user-by-email db "a@test.com")

;; Pull API: retrieve entity as map
(d/pull db '[:user/name :user/email :user/age] [:user/email "a@test.com"])
;; => {:user/name "สมชาย", :user/email "a@test.com", :user/age 25}

;; Pull with nested refs
(d/pull db '[* {:order/user [:user/name :user/email]}]
        [:db/id 1000])
```

---

## ขั้นตอนที่ 605: Time Travel

```clojure
;; Datomic เก็บทุก transaction → time travel ได้!

;; ดู database ณ เวลาในอดีต
(def old-db (d/as-of db (java.util.Date. 1704067200000)))  ; Jan 1 2024

;; ดูข้อมูลตอนนั้น
(d/pull old-db '[:user/name :user/age] [:user/email "a@test.com"])

;; ดูประวัติ attribute
(d/q '[:find ?tx ?added ?age
       :in $ ?e
       :where
       [?e :user/age ?age ?tx ?added]]
     (d/history db)  ; history database
     [:user/email "a@test.com"])
;; => #{[1000 true 25]   ; added age 25 in tx 1000
;;      [2000 false 25]  ; retracted age 25 in tx 2000
;;      [2000 true 26]}  ; asserted age 26 in tx 2000

;; Database as-of specific transaction
(def db-at-tx1000 (d/as-of db 1000))
(d/pull db-at-tx1000 '[:user/age] [:user/email "a@test.com"])
;; => {:user/age 25}   (before the update)

(def db-at-tx2000 (d/as-of db 2000))
(d/pull db-at-tx2000 '[:user/age] [:user/email "a@test.com"])
;; => {:user/age 26}   (after the update)
```

---

## ขั้นตอนที่ 606: Advanced Queries

```clojure
;; Aggregates
(d/q '[:find (count ?e) (avg ?age)
       :with ?e
       :where
       [?e :user/age ?age]]
     db)
;; => [[2 27.5]]

;; Rules: reusable query predicates
(def rules
  '[[(adult? ?e)
     [?e :user/age ?age]
     [(>= ?age 18)]]
    
    [(admin? ?e)
     [?e :user/roles :admin]]])

(d/q '[:find ?name
       :in $ %
       :where
       [?e :user/name ?name]
       (adult? ?e)
       (admin? ?e)]
     db rules)

;; Full-text search
(d/q '[:find ?name ?email
       :where
       [(fulltext $ :user/name "สม") [[?e ?name]]]
       [?e :user/email ?email]]
     db)

;; Recursive query: find all descendants
(def ancestry-rules
  '[[(ancestor? ?a ?b)
     [?b :person/parent ?a]]
    [(ancestor? ?a ?b)
     [?b :person/parent ?c]
     (ancestor? ?a ?c)]])

(d/q '[:find ?ancestor-name
       :in $ % ?person
       :where
       (ancestor? ?ancestor ?person)
       [?ancestor :person/name ?ancestor-name]]
     db ancestry-rules person-id)
```

---

## ขั้นตอนที่ 607: Entity API

```clojure
;; Entity: lazy map-like view of Datomic entity

(def user-entity (d/entity db [:user/email "a@test.com"]))

;; Access attributes
(:user/name user-entity)   ; => "สมชาย"
(:user/age user-entity)    ; => 25
(:user/roles user-entity)  ; => #{:admin :user}

;; Navigate references
(def order-entity (d/entity db order-id))
(-> order-entity :order/user :user/name)  ; traverse ref!

;; Touch: realize all attributes
(d/touch user-entity)
;; => {:db/id 100, :user/name "สมชาย", :user/email "...", :user/age 25, ...}

;; Back references: reverse navigation
;; If :order/user is a ref TO user,
;; user can access ._order/user (reverse ref)
(-> (d/entity db user-id)
    :order/_user)  ; all orders where user is the user
```

---

## Project: Inventory System ด้วย Datomic

```clojure
(ns inventory.core
  (:require [datomic.api :as d]))

;; Schema
(def schema
  [{:db/ident :product/id    :db/valueType :db.type/uuid   :db/cardinality :db.cardinality/one :db/unique :db.unique/identity}
   {:db/ident :product/name  :db/valueType :db.type/string :db/cardinality :db.cardinality/one}
   {:db/ident :product/sku   :db/valueType :db.type/string :db/cardinality :db.cardinality/one :db/unique :db.unique/value}
   {:db/ident :product/price :db/valueType :db.type/bigdec :db/cardinality :db.cardinality/one}
   {:db/ident :product/stock :db/valueType :db.type/long   :db/cardinality :db.cardinality/one}
   
   {:db/ident :tx/reason     :db/valueType :db.type/string :db/cardinality :db.cardinality/one}])

;; Add product
(defn add-product! [conn product]
  @(d/transact conn [(assoc product :db/id "new-product")]))

;; Update stock ด้วย transaction annotation
(defn update-stock! [conn sku delta reason]
  (let [db      (d/db conn)
        entity  (d/entity db [:product/sku sku])
        current (:product/stock entity)
        new-qty (+ current delta)]
    (when (< new-qty 0)
      (throw (ex-info "Insufficient stock" {:sku sku :current current :delta delta})))
    @(d/transact conn
       [;; Update stock
        {:db/id         [:product/sku sku]
         :product/stock new-qty}
        ;; Annotate the transaction itself
        {:db/id "datomic.tx"
         :tx/reason reason}])))

;; View stock history (time travel!)
(defn stock-history [db sku]
  (d/q '[:find ?tx ?added ?stock ?reason
         :in $ ?sku
         :where
         [?e :product/sku ?sku]
         [?e :product/stock ?stock ?tx ?added]
         [(get-else $ ?tx :tx/reason "no-reason") ?reason]]
       (d/history db) sku))

;; ใช้งาน
(comment
  (def conn (d/connect "datomic:mem://inventory"))
  @(d/transact conn schema)
  
  (add-product! conn
    {:product/id    (java.util.UUID/randomUUID)
     :product/name  "Widget A"
     :product/sku   "WDGT-001"
     :product/price 99.99M
     :product/stock 100})
  
  (update-stock! conn "WDGT-001" -10 "Order ORD-001")
  (update-stock! conn "WDGT-001" -5  "Order ORD-002")
  (update-stock! conn "WDGT-001" 50  "Restock from supplier")
  
  (stock-history (d/db conn) "WDGT-001")
  ;; See full history!
  )
```

---

*Part 21 จาก 100+ | ขั้นตอน 601-630 จาก 1000+*
