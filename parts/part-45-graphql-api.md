# Part 45: GraphQL API ด้วย Lacinia
## ขั้นตอนที่ 1321-1350: Schema Definition, Resolvers, Subscriptions, DataLoader

---

## บทนำ

GraphQL ด้วย Lacinia (Clojure GraphQL library):
- **Schema Definition** - types, queries, mutations
- **Resolvers** - data fetching functions
- **N+1 Problem** - DataLoader pattern
- **Subscriptions** - real-time GraphQL over WebSocket
- **Authorization** - field-level security

---

## ขั้นตอนที่ 1321: Lacinia Schema Definition

```clojure
;; deps.edn
;; {:deps {com.walmartlabs/lacinia {:mvn/version "1.2.2"}
;;         com.walmartlabs/lacinia-pedestal {:mvn/version "1.2.2"}}}

(ns myapp.schema)

(def schema
  {:enums
   {:OrderStatus
    {:values [:PENDING :CONFIRMED :SHIPPED :DELIVERED :CANCELLED]}
    
    :UserRole
    {:values [:GUEST :USER :SELLER :ADMIN]}}
   
   :interfaces
   {:Node
    {:fields {:id {:type '(non-null ID)}}}}
   
   :objects
   {:User
    {:implements [:Node]
     :fields
     {:id         {:type '(non-null ID)}
      :name       {:type '(non-null String)}
      :email      {:type '(non-null String)}
      :role       {:type :UserRole}
      :orders     {:type '(list :Order)
                    :resolve :resolve-user-orders}
      :orderCount {:type Int
                    :resolve :resolve-user-order-count}
      :createdAt  {:type String}}}
    
    :Product
    {:implements [:Node]
     :fields
     {:id          {:type '(non-null ID)}
      :name        {:type '(non-null String)}
      :description {:type String}
      :price       {:type '(non-null Float)}
      :category    {:type String}
      :stock       {:type Int}
      :seller      {:type :User
                     :resolve :resolve-product-seller}
      :reviews     {:type '(list :Review)
                     :resolve :resolve-product-reviews}
      :avgRating   {:type Float
                     :resolve :resolve-product-avg-rating}}}
    
    :Order
    {:implements [:Node]
     :fields
     {:id        {:type '(non-null ID)}
      :status    {:type '(non-null :OrderStatus)}
      :total     {:type '(non-null Float)}
      :items     {:type '(non-null (list (non-null :OrderItem)))}
      :user      {:type :User
                   :resolve :resolve-order-user}
      :createdAt {:type String}}}
    
    :OrderItem
    {:fields
     {:product  {:type '(non-null :Product)
                  :resolve :resolve-order-item-product}
      :quantity {:type '(non-null Int)}
      :price    {:type '(non-null Float)}}}
    
    :Review
    {:fields
     {:id      {:type '(non-null ID)}
      :rating  {:type '(non-null Int)}
      :comment {:type String}
      :user    {:type :User
                 :resolve :resolve-review-user}}}
    
    :PageInfo
    {:fields
     {:hasNextPage     {:type '(non-null Boolean)}
      :hasPreviousPage {:type '(non-null Boolean)}
      :startCursor     {:type String}
      :endCursor       {:type String}}}
    
    :ProductConnection
    {:fields
     {:edges    {:type '(list :ProductEdge)}
      :pageInfo {:type '(non-null :PageInfo)}
      :total    {:type Int}}}
    
    :ProductEdge
    {:fields
     {:node   {:type :Product}
      :cursor {:type String}}}}
   
   :input-objects
   {:CreateProductInput
    {:fields
     {:name        {:type '(non-null String)}
      :description {:type String}
      :price       {:type '(non-null Float)}
      :category    {:type String}
      :stock       {:type Int}}}
    
    :PlaceOrderInput
    {:fields
     {:items   {:type '(non-null (list (non-null :OrderItemInput)))}
      :address {:type '(non-null :AddressInput)}}}
    
    :OrderItemInput
    {:fields
     {:productId {:type '(non-null ID)}
      :quantity  {:type '(non-null Int)}}}
    
    :AddressInput
    {:fields
     {:street   {:type '(non-null String)}
      :city     {:type '(non-null String)}
      :province {:type '(non-null String)}
      :zipCode  {:type '(non-null String)}}}}
   
   :queries
   {:user
    {:type :User
     :args {:id {:type '(non-null ID)}}
     :resolve :resolve-user}
    
    :me
    {:type :User
     :resolve :resolve-me}
    
    :product
    {:type :Product
     :args {:id {:type '(non-null ID)}}
     :resolve :resolve-product}
    
    :products
    {:type :ProductConnection
     :args {:first    {:type Int}
             :after    {:type String}
             :category {:type String}
             :query    {:type String}}
     :resolve :resolve-products}
    
    :order
    {:type :Order
     :args {:id {:type '(non-null ID)}}
     :resolve :resolve-order}
    
    :myOrders
    {:type '(list :Order)
     :resolve :resolve-my-orders}}
   
   :mutations
   {:createProduct
    {:type :Product
     :args {:input {:type '(non-null :CreateProductInput)}}
     :resolve :mutate-create-product}
    
    :placeOrder
    {:type :Order
     :args {:input {:type '(non-null :PlaceOrderInput)}}
     :resolve :mutate-place-order}
    
    :cancelOrder
    {:type :Order
     :args {:orderId {:type '(non-null ID)}}
     :resolve :mutate-cancel-order}}
   
   :subscriptions
   {:orderStatusChanged
    {:type :Order
     :args {:orderId {:type '(non-null ID)}}
     :stream :stream-order-status}}})
```

---

## ขั้นตอนที่ 1322: Resolvers

```clojure
(ns myapp.resolvers
  (:require [com.walmartlabs.lacinia :as lacinia]
            [next.jdbc :as jdbc]))

;; Base resolver context
(defn create-context [db redis user]
  {:db    db
   :redis redis
   :user  user})

;; Query resolvers
(defn resolve-user [context args _value]
  (let [{:keys [id]} args
        {:keys [db]} context]
    (jdbc/execute-one! db
      ["SELECT * FROM users WHERE id = ?" (java.util.UUID/fromString id)])))

(defn resolve-me [context _args _value]
  (when-let [user-id (get-in context [:user :uid])]
    (jdbc/execute-one! (:db context)
      ["SELECT * FROM users WHERE id = ?" (java.util.UUID/fromString user-id)])))

(defn resolve-product [context args _value]
  (let [{:keys [id]} args]
    (jdbc/execute-one! (:db context)
      ["SELECT * FROM products WHERE id = ?" (java.util.UUID/fromString id)])))

;; Pagination with cursor
(defn resolve-products [context args _value]
  (let [{:keys [first after category query]} args
        limit   (min (or first 20) 100)
        offset  (if after (cursor->offset after) 0)
        
        where   (cond-> ["status = 'active'"]
                  category (conj "category = ?")
                  query    (conj "to_tsvector('english', name) @@ plainto_tsquery('english', ?)"))
        params  (cond-> []
                  category (conj category)
                  query    (conj query))
        
        total   (first (jdbc/execute-one! (:db context)
                          (into [(str "SELECT COUNT(*) FROM products WHERE "
                                       (clojure.string/join " AND " where))]
                                params)))
        items   (jdbc/execute! (:db context)
                  (into [(str "SELECT * FROM products WHERE "
                               (clojure.string/join " AND " where)
                               " LIMIT ? OFFSET ?")]
                        (conj params limit offset)))]
    
    {:edges    (map #(hash-map :node % :cursor (offset->cursor (+ offset %2))) items (range))
     :pageInfo {:hasNextPage     (> (- total offset) limit)
                 :hasPreviousPage (> offset 0)
                 :startCursor    (offset->cursor offset)
                 :endCursor      (offset->cursor (+ offset (count items) -1))}
     :total    total}))

;; Mutation resolvers
(defn mutate-place-order [context args _value]
  (let [{:keys [input]} args
        user-id (get-in context [:user :uid])]
    (when-not user-id
      (throw (lacinia/resolve-as nil {:message "Authentication required"})))
    
    (order-service/place-order!
      (:db context)
      user-id
      (:items input)
      (:address input))))
```

---

## ขั้นตอนที่ 1323: DataLoader สำหรับ N+1 Problem

```clojure
(ns myapp.dataloader
  (:require [com.walmartlabs.lacinia :as lacinia]))

;; DataLoader: batch and cache DB calls
(defn create-batch-fn [db query key-field]
  (fn [ids]
    (let [results (jdbc/execute! db
                    [(str "SELECT * FROM " (name (get-table-name query))
                           " WHERE id IN (" (clojure.string/join "," (repeat (count ids) "?")) ")")
                     ids])
          by-id   (group-by (keyword key-field) results)]
      (map #(first (get by-id %)) ids))))

;; User loader
(defn user-loader [db]
  (memoize
    (let [cache    (atom {})
          pending  (atom [])]
      (fn [user-id]
        (if-let [cached (get @cache user-id)]
          cached
          (do
            (swap! pending conj user-id)
            ;; Batch load on next tick
            (let [ids    @pending
                  _      (reset! pending [])
                  users  (jdbc/execute! db
                           [(str "SELECT * FROM users WHERE id IN ("
                                  (clojure.string/join "," (repeat (count ids) "?"))
                                  ")")
                            ids])]
              (doseq [u users]
                (swap! cache assoc (:id u) u))
              (get @cache user-id))))))))

;; Register resolvers with DataLoader
(defn create-resolvers [db]
  (let [user-load   (user-loader db)
        product-load (product-loader db)]
    {:resolve-review-user
     (fn [_ctx _args review]
       (user-load (:user_id review)))
     
     :resolve-order-user
     (fn [_ctx _args order]
       (user-load (:user_id order)))
     
     :resolve-order-item-product
     (fn [_ctx _args item]
       (product-load (:product_id item)))}))
```

---

## ขั้นตอนที่ 1324: GraphQL Subscriptions

```clojure
(ns myapp.subscriptions
  (:require [clojure.core.async :as async]
            [com.walmartlabs.lacinia.async :as lacinia-async]))

;; Subscription streams
(def order-update-channels (atom {}))  ; order-id -> [channels]

(defn stream-order-status [context args source-stream]
  (let [order-id (:orderId args)
        ch       (async/chan 10)]
    
    ;; Register channel
    (swap! order-update-channels update order-id (fnil conj []) ch)
    
    ;; Stream function: called by lacinia per message
    (fn []
      (async/go
        (when-let [order (async/<! ch)]
          (source-stream order))))
    
    ;; Cleanup on unsubscribe
    (fn []
      (swap! order-update-channels update order-id
             (fnil #(remove #{ch} %) []))
      (async/close! ch))))

;; Publish order updates
(defn publish-order-update! [order]
  (doseq [ch (get @order-update-channels (str (:id order)) [])]
    (async/put! ch order)))

;; Setup Pedestal + Lacinia
(defn create-app [schema db]
  (let [compiled (lacinia/compile schema {:resolvers (create-resolvers db)})]
    (com.walmartlabs.lacinia.pedestal2/default-service compiled
      {:port            8080
       :app-context-fn  (fn [request]
                          {:db    db
                           :user  (extract-user request)})})))
```

---

## Project: GraphQL Shopping API

```clojure
(ns shop.graphql
  (:require [com.walmartlabs.lacinia :as lacinia]
            [myapp.schema :as schema]
            [myapp.resolvers :as resolvers]))

;; Assemble complete GraphQL service
(defn create-graphql-service [db redis]
  (let [compiled-schema
        (lacinia/compile
          schema/schema
          {:resolvers
           (merge
             ;; Query resolvers
             {:resolve-user          (resolvers/resolve-user db)
              :resolve-me            (resolvers/resolve-me)
              :resolve-product       (resolvers/resolve-product db redis)
              :resolve-products      (resolvers/resolve-products db)
              :resolve-order         (resolvers/resolve-order db)
              :resolve-my-orders     (resolvers/resolve-my-orders db)}
             
             ;; Relationship resolvers (with DataLoader)
             (resolvers/create-relationship-resolvers db)
             
             ;; Mutations
             {:mutate-create-product (resolvers/mutate-create-product db)
              :mutate-place-order    (resolvers/mutate-place-order db redis)
              :mutate-cancel-order   (resolvers/mutate-cancel-order db)}
             
             ;; Subscriptions
             {:stream-order-status   subscriptions/stream-order-status})})]
    
    (com.walmartlabs.lacinia.pedestal2/default-service compiled-schema
      {:port   8080
       :host   "0.0.0.0"
       :app-context-fn
       (fn [request]
         {:db    db
          :redis redis
          :user  (auth/extract-claims request)})})))
```

---

*Part 45 จาก 100+ | ขั้นตอน 1321-1350 จาก 1000+*
