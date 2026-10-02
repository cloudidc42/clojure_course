# Part 19: GraphQL ด้วย Lacinia
## ขั้นตอนที่ 541-570: Schema Definition, Resolvers, Subscriptions

---

## บทนำ

**Lacinia** คือ GraphQL implementation สำหรับ Clojure
- Schema ใช้ EDN (Clojure data)
- Resolvers คือ functions
- เหมาะกับ Clojure's data-oriented approach

---

## ขั้นตอนที่ 541: GraphQL Schema

```clojure
;; deps.edn
;; {:deps {com.walmartlabs/lacinia {:mvn/version "1.2.2"}
;;         com.walmartlabs/lacinia-pedestal {:mvn/version "1.2.1"}}}

;; Schema ใน EDN
(def schema
  {:objects
   {:User
    {:fields
     {:id       {:type '(non-null ID)}
      :name     {:type '(non-null String)}
      :email    {:type '(non-null String)}
      :age      {:type 'Int}
      :role     {:type :Role}
      :orders   {:type '(list :Order)
                 :description "User's orders"}}}
    
    :Order
    {:fields
     {:id         {:type '(non-null ID)}
      :status     {:type :OrderStatus}
      :total      {:type 'Float}
      :items      {:type '(list :OrderItem)}
      :created-at {:type 'String}}}
    
    :OrderItem
    {:fields
     {:product  {:type :Product}
      :qty      {:type 'Int}
      :price    {:type 'Float}}}
    
    :Product
    {:fields
     {:id    {:type '(non-null ID)}
      :name  {:type '(non-null String)}
      :price {:type 'Float}}}}
   
   :enums
   {:Role        {:values [:ADMIN :USER :MODERATOR]}
    :OrderStatus {:values [:PENDING :CONFIRMED :SHIPPED :DELIVERED :CANCELLED]}}
   
   :queries
   {:user     {:type        :User
               :description "Get user by ID"
               :args        {:id {:type '(non-null ID)}}
               :resolve     :query/user}
    
    :users    {:type    '(list :User)
               :args    {:page {:type 'Int :default-value 1}
                          :size {:type 'Int :default-value 20}}
               :resolve :query/users}
    
    :product  {:type    :Product
               :args    {:id {:type '(non-null ID)}}
               :resolve :query/product}}
   
   :mutations
   {:create-user   {:type    :User
                    :args    {:name  {:type '(non-null String)}
                               :email {:type '(non-null String)}}
                    :resolve :mutation/create-user}
    
    :place-order   {:type    :Order
                    :args    {:customer-id {:type '(non-null ID)}
                               :items {:type '(non-null (list :OrderItemInput))}}
                    :resolve :mutation/place-order}}
   
   :input-objects
   {:OrderItemInput
    {:fields {:product-id {:type '(non-null ID)}
               :qty        {:type '(non-null Int)}}}}})
```

---

## ขั้นตอนที่ 542: Resolvers

```clojure
(ns myapp.graphql.resolvers
  (:require [lacinia.util :as util]
            [myapp.db :as db]))

;; Resolver = function: context, args, value → result

(defn resolve-user [context {:keys [id]} _value]
  (db/find-user (:db context) (java.util.UUID/fromString id)))

(defn resolve-users [context {:keys [page size]} _value]
  (db/find-users (:db context) {:page page :size size}))

(defn resolve-user-orders [context _args user]
  (db/find-orders-by-user (:db context) (:id user)))

(defn resolve-create-user [context {:keys [name email]} _value]
  (let [user {:name name :email email}]
    ;; Validate
    (when-not (re-matches #".+@.+\..+" email)
      (util/throw-error "Invalid email" {:email email}))
    ;; Create
    (db/create-user! (:db context) user)))

(defn resolve-place-order [context {:keys [customer-id items]} _value]
  (let [order-items (map #(assoc % :product-id
                            (java.util.UUID/fromString (:product-id %)))
                         items)]
    (order-service/place-order! (:deps context)
      {:customer-id (java.util.UUID/fromString customer-id)
       :items order-items})))

;; Resolver map
(def resolvers
  {:query/user            resolve-user
   :query/users           resolve-users
   :mutation/create-user  resolve-create-user
   :mutation/place-order  resolve-place-order
   :User/orders           resolve-user-orders})
```

---

## ขั้นตอนที่ 543: GraphQL Server Setup

```clojure
(ns myapp.graphql.server
  (:require [com.walmartlabs.lacinia :as lacinia]
            [com.walmartlabs.lacinia.schema :as schema]
            [com.walmartlabs.lacinia.util :as util]
            [com.walmartlabs.lacinia.pedestal2 :as lp]
            [io.pedestal.http :as http]))

;; Compile schema with resolvers
(defn create-schema []
  (-> schema-definition
      (util/attach-resolvers resolvers)
      schema/compile))

;; Create Pedestal service
(defn create-service [db]
  (let [compiled-schema (create-schema)
        app-context {:db db}]
    (-> {:schema         compiled-schema
         :app-context    app-context
         :port           8888
         :env            :dev}
        lp/default-service
        http/create-server)))

;; Execute query programmatically
(defn execute [schema query variables context]
  (lacinia/execute schema query variables context))

;; Test
(comment
  (def result
    (execute (create-schema)
             "{ user(id: \"abc\") { name email } }"
             nil
             {:db test-db}))
  
  ;; => {:data {:user {:name "สมชาย" :email "test@test.com"}}}
  )
```

---

## ขั้นตอนที่ 544: N+1 Problem และ DataLoader

```clojure
;; N+1 Problem: query orders → สำหรับทุก order ดึง user แยกกัน
;; Users query → 1 DB call
;; Orders per user → N DB calls (หนึ่งครั้งต่อ user!)

;; Solution: DataLoader (batch loading)
;; Lacinia มี built-in support สำหรับ batching

(ns myapp.graphql.loaders
  (:require [com.walmartlabs.lacinia.executor :as executor]))

;; Batch loader: receive multiple IDs, return all at once
(defn batch-load-users [db user-ids]
  (let [users (db/find-users-by-ids db user-ids)]
    ;; Must return map: id → value
    (into {} (map (juxt :id identity) users))))

;; Use in resolver
(defn resolve-order-user [context _args order]
  (executor/selectively-resolve
    context
    :user-loader
    (:user-id order)))  ; queued for batching!

;; Register loaders
(def loaders
  {:user-loader (fn [context ids]
                   (batch-load-users (:db context) ids))})

;; All user loads in a single request are batched automatically!
;; 1 request with 100 orders → 1 DB query (not 100!)
```

---

## ขั้นตอนที่ 545: GraphQL Authentication

```clojure
;; Authentication via middleware
(defn graphql-auth-middleware [handler]
  (fn [request]
    (let [token (some-> (get-in request [:headers "authorization"])
                        (clojure.string/replace #"^Bearer " ""))
          claims (when token (auth/verify-token token))]
      (handler (assoc-in request [:lacinia-app-context :current-user]
                         claims)))))

;; Authorization in resolvers
(defn resolve-admin-only [context args value]
  (let [user (:current-user context)]
    (when-not (and user (= :ADMIN (:role user)))
      (util/throw-error "Forbidden" {:code :FORBIDDEN}))
    ;; Proceed with resolution
    (get-admin-data args)))

;; Field-level authorization
(defn authorized-field [auth-fn resolver]
  (fn [context args value]
    (auth-fn context)
    (resolver context args value)))

;; Directive-based auth
;; @auth(requires: ADMIN)
```

---

## Project: GraphQL API สำหรับ Blog

```clojure
(def blog-schema
  {:objects
   {:Post
    {:fields {:id       {:type '(non-null ID)}
               :title    {:type '(non-null String)}
               :content  {:type 'String}
               :author   {:type :Author}
               :tags     {:type '(list String)}
               :comments {:type '(list :Comment)}
               :created-at {:type 'String}}}
    
    :Author
    {:fields {:id    {:type '(non-null ID)}
               :name  {:type '(non-null String)}
               :bio   {:type 'String}
               :posts {:type '(list :Post)}}}
    
    :Comment
    {:fields {:id      {:type '(non-null ID)}
               :content {:type '(non-null String)}
               :author  {:type :Author}
               :created-at {:type 'String}}}}
   
   :queries
   {:posts       {:type '(list :Post)
                  :args {:tag {:type 'String}
                          :page {:type 'Int}}
                  :resolve :query/posts}
    
    :post        {:type :Post
                  :args {:id {:type '(non-null ID)}}
                  :resolve :query/post}
    
    :search-posts {:type '(list :Post)
                   :args {:query {:type '(non-null String)}}
                   :resolve :query/search-posts}}
   
   :mutations
   {:create-post   {:type :Post
                    :args {:title   {:type '(non-null String)}
                            :content {:type '(non-null String)}
                            :tags    {:type '(list String)}}
                    :resolve :mutation/create-post}
    
    :add-comment   {:type :Comment
                    :args {:post-id {:type '(non-null ID)}
                            :content {:type '(non-null String)}}
                    :resolve :mutation/add-comment}
    
    :delete-post   {:type :Post
                    :args {:id {:type '(non-null ID)}}
                    :resolve :mutation/delete-post}}})

;; Example queries
(comment
  ;; Get all posts with specific tag
  "{
    posts(tag: \"clojure\", page: 1) {
      id
      title
      author { name }
      tags
      comments { content }
    }
  }"
  
  ;; Create post mutation
  "mutation CreatePost($title: String!, $content: String!) {
    createPost(title: $title, content: $content) {
      id
      title
      createdAt
    }
  }")
```

---

*Part 19 จาก 100+ | ขั้นตอน 541-570 จาก 1000+*
