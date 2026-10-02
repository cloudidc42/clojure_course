# Part 72: GraphQL ด้วย Clojure
## ขั้นตอนที่ 2131-2160: Schema, Resolvers, DataLoader, Subscriptions, Auth

---

## บทนำ

GraphQL ใน Clojure ด้วย Lacinia:
- **Schema definition** - SDL หรือ EDN format
- **Resolvers** - fetch data ตาม field
- **DataLoader** - แก้ N+1 problem
- **Subscriptions** - real-time updates
- **Mutations** - write operations

---

## ขั้นตอนที่ 2131: Schema Definition

```clojure
(ns myapp.graphql.schema
  (:require [com.walmartlabs.lacinia.schema :as schema]
            [com.walmartlabs.lacinia.util   :as util]))

;; Define schema in EDN
(def schema-edn
  {:enums
   {:OrderStatus
    {:values [:PENDING :PROCESSING :SHIPPED :DELIVERED :CANCELLED]}}
   
   :objects
   {:Product
    {:description "A product in the catalog"
     :fields
     {:id          {:type (non-null :ID)}
      :name        {:type (non-null :String)}
      :description {:type :String}
      :price       {:type (non-null :Float)}
      :stock       {:type (non-null :Int)}
      :category    {:type :Category}
      :reviews     {:type (list :Review)
                    :args {:limit {:type :Int :default-value 10}}}}}
    
    :Category
    {:fields
     {:id       {:type (non-null :ID)}
      :name     {:type (non-null :String)}
      :products {:type (list :Product)}}}
    
    :Order
    {:fields
     {:id         {:type (non-null :ID)}
      :status     {:type (non-null :OrderStatus)}
      :total      {:type (non-null :Float)}
      :items      {:type (non-null (list :OrderItem))}
      :customer   {:type (non-null :Customer)}
      :created_at {:type (non-null :String)}}}
    
    :OrderItem
    {:fields
     {:product  {:type (non-null :Product)}
      :quantity {:type (non-null :Int)}
      :price    {:type (non-null :Float)}}}
    
    :Customer
    {:fields
     {:id     {:type (non-null :ID)}
      :name   {:type (non-null :String)}
      :email  {:type (non-null :String)}
      :orders {:type (list :Order)
               :args {:page     {:type :Int :default-value 1}
                      :pageSize {:type :Int :default-value 10}}}}}}
   
   :queries
   {:product
    {:type    :Product
     :args    {:id {:type (non-null :ID)}}
     :resolve :query/product}
    
    :products
    {:type    (list :Product)
     :args    {:category {:type :String}
               :search   {:type :String}
               :page     {:type :Int :default-value 1}
               :pageSize {:type :Int :default-value 20}}
     :resolve :query/products}
    
    :order
    {:type    :Order
     :args    {:id {:type (non-null :ID)}}
     :resolve :query/order}}
   
   :mutations
   {:createOrder
    {:type    :Order
     :args    {:input {:type (non-null :CreateOrderInput)}}
     :resolve :mutation/create-order}
    
    :updateOrderStatus
    {:type    :Order
     :args    {:id     {:type (non-null :ID)}
               :status {:type (non-null :OrderStatus)}}
     :resolve :mutation/update-order-status}}
   
   :subscriptions
   {:orderUpdated
    {:type    :Order
     :args    {:orderId {:type (non-null :ID)}}
     :stream  :subscription/order-updated}}
   
   :input-objects
   {:CreateOrderInput
    {:fields
     {:customerId {:type (non-null :ID)}
      :items      {:type (non-null (list :OrderItemInput))}}}
    
    :OrderItemInput
    {:fields
     {:productId {:type (non-null :ID)}
      :quantity  {:type (non-null :Int)}}}}})
```

---

## ขั้นตอนที่ 2132: Resolvers

```clojure
(ns myapp.graphql.resolvers
  (:require [com.walmartlabs.lacinia :as lacinia]))

;; Query resolvers
(defn resolve-product [db]
  (fn [context args _parent]
    (let [{:keys [id]} args]
      (when-let [product (db/get-product db id)]
        (assoc product :_type :Product)))))

(defn resolve-products [db search-engine]
  (fn [context {:keys [category search page pageSize]} _parent]
    (cond
      search   (search-engine/search search-engine
                 {:q        search
                  :category category
                  :page     page
                  :size     pageSize})
      category (db/get-products-by-category db category page pageSize)
      :else    (db/list-products db page pageSize))))

;; Field resolvers (for nested data)
(defn resolve-product-category [db]
  (fn [_context _args product]
    (db/get-category db (:category-id product))))

(defn resolve-product-reviews [db]
  (fn [_context {:keys [limit]} product]
    (db/get-product-reviews db (:id product) limit)))

(defn resolve-order-customer [db]
  (fn [_context _args order]
    (db/get-customer db (:customer-id order))))

;; Mutations
(defn resolve-create-order [db]
  (fn [context {:keys [input]} _parent]
    ;; Check auth
    (when-not (get-in context [:user :id])
      (throw (ex-info "Unauthorized" {:code :unauthorized})))
    
    (let [{:keys [customerId items]} input]
      (order/create-order! db
        {:customer-id customerId
         :items       (map (fn [{:keys [productId quantity]}]
                              {:product-id productId
                               :quantity   quantity})
                            items)
         :created-by  (get-in context [:user :id])}))))

;; Assemble resolvers map
(defn make-resolver-map [db search-engine]
  {:query/product         (resolve-product db)
   :query/products        (resolve-products db search-engine)
   :query/order           (resolve-order db)
   :mutation/create-order (resolve-create-order db)
   
   ;; Field resolvers
   :Product/category (resolve-product-category db)
   :Product/reviews  (resolve-product-reviews db)
   :Order/customer   (resolve-order-customer db)})
```

---

## ขั้นตอนที่ 2133: DataLoader (N+1 Prevention)

```clojure
(ns myapp.graphql.dataloader
  (:require [promesa.core :as p]))

;; DataLoader batches multiple resolver calls into one DB query
;; Problem: 100 products each resolving their category = 100 queries
;; Solution: batch into 1 query with all category IDs

(defprotocol DataLoader
  (load! [this key])
  (dispatch! [this]))

(defn make-batch-loader [batch-fn]
  (let [pending   (atom [])
        promises  (atom {})
        scheduled (atom false)]
    
    (reify DataLoader
      (load! [this key]
        (let [promise (p/deferred)]
          (swap! pending conj key)
          (swap! promises assoc key promise)
          (when (not @scheduled)
            (reset! scheduled true)
            ;; Schedule microtask to dispatch
            (p/then (p/resolved nil)
              (fn [_]
                (dispatch! this))))
          promise))
      
      (dispatch! [this]
        (let [keys     @pending
              all-prms @promises]
          (reset! pending [])
          (reset! promises {})
          (reset! scheduled false)
          
          (p/then (batch-fn keys)
            (fn [results]
              (doseq [[k v] results]
                (when-let [p (get all-prms k)]
                  (p/resolve! p v))))))))))

;; Category DataLoader
(defn make-category-loader [db]
  (make-batch-loader
    (fn [category-ids]
      (let [rows (jdbc/execute! db
                   (into ["SELECT * FROM categories WHERE id = ANY(?::uuid[])"]
                         [(into-array String category-ids)]))]
        (into {} (map #(vector (str (:categories/id %)) %) rows))))))

;; Use in resolver
(defn resolve-product-category-batched [db]
  (fn [context _args product]
    (let [loader (or (get-in context [:loaders :category])
                     (make-category-loader db))]
      (load! loader (:category-id product)))))

;; Create loaders per request (to avoid cross-request caching)
(defn make-loaders [db]
  {:category (make-category-loader db)
   :customer (make-customer-loader db)
   :product  (make-product-loader db)})
```

---

## ขั้นตอนที่ 2134: Subscriptions

```clojure
(ns myapp.graphql.subscriptions
  (:require [clojure.core.async :as async]))

;; Subscription source
(defn order-updated-stream [context {:keys [orderId]} _source-stream]
  (let [event-ch (async/chan 100)
        cleanup  (subscribe-to-order-events! orderId
                   (fn [event]
                     (async/put! event-ch {:type :order-updated
                                           :order event})))]
    ;; Return cleanup function
    (fn []
      (cleanup)
      (async/close! event-ch))))

;; Subscription handler using SSE
(defn graphql-subscription-handler [compiled-schema redis-pool request]
  (let [{:keys [query variables]} (:body-params request)
        user-id (get-in request [:user :id])]
    
    ;; Server-Sent Events for subscriptions
    {:status  200
     :headers {"Content-Type"  "text/event-stream"
               "Cache-Control" "no-cache"
               "Connection"    "keep-alive"}
     :body
     (reify ring.core.protocols/StreamableResponseBody
       (write-body-to-stream [_ _ output-stream]
         (let [writer   (java.io.PrintWriter. output-stream true)
               channels (atom [])
               stop!    (fn []
                           (doseq [ch @channels] (async/close! ch))
                           (.close writer))]
           (try
             (lacinia/execute-streaming
               compiled-schema query variables
               {:user    {:id user-id}
                :stop-fn stop!}
               (fn [result]
                 (.println writer (str "data: " (json/generate-string result)))
                 (.println writer "")
                 (.flush writer)))
             (catch Exception e
               (stop!))))))}))
```

---

## ขั้นตอนที่ 2135: GraphQL Auth and Error Handling

```clojure
;; Per-field authorization
(defn authorized-field [permission resolver-fn]
  (fn [context args parent]
    (when-not (has-permission? (get-in context [:user :roles]) permission)
      (throw (ex-info "Unauthorized"
                       {:code     :unauthorized
                        :message  "Insufficient permissions"
                        :path     (:path context)})))
    (resolver-fn context args parent)))

;; Error formatting
(defn format-graphql-error [error]
  (let [data (ex-data error)]
    {:message   (.getMessage error)
     :locations []
     :extensions (case (:code data)
                    :unauthorized {:code "UNAUTHORIZED" :status 401}
                    :not-found    {:code "NOT_FOUND" :status 404}
                    :validation   {:code "VALIDATION_ERROR"
                                   :fields (:fields data)}
                    {:code "INTERNAL_ERROR"})}))

;; Complete GraphQL handler
(defn build-graphql-handler [db search-engine redis-pool]
  (let [resolver-map  (make-resolver-map db search-engine)
        compiled      (-> schema-edn
                          (util/attach-resolvers resolver-map)
                          schema/compile)]
    
    (fn [request]
      (let [{:keys [query variables operation-name]} (:body-params request)
            context {:user     (:user request)
                     :db       db
                     :loaders  (make-loaders db)
                     :redis    redis-pool}
            result  (lacinia/execute compiled query variables context
                      {:operation-name operation-name})]
        {:status  (if (seq (:errors result)) 200 200)  ; GraphQL always 200
         :body    result}))))
```

---

*Part 72 จาก 100+ | ขั้นตอน 2131-2160 จาก 1000+*
