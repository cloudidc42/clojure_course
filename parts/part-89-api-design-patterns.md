# Part 89: API Design Patterns ขั้นสูง
## ขั้นตอนที่ 2641-2670: REST Maturity, Hypermedia, Versioning, Rate Limiting, Pagination

---

## บทนำ

API design ระดับ world-class:
- **Richardson Maturity Model** - levels of REST
- **Hypermedia/HATEOAS** - self-describing APIs
- **API Versioning** - header vs URL vs content-type
- **Cursor pagination** - efficient large datasets
- **Rate limiting** - token bucket per user/endpoint

---

## ขั้นตอนที่ 2641: REST Maturity Level 3 - Hypermedia

```clojure
(ns myapp.api.hypermedia)

;; HAL (Hypertext Application Language) format
(defn hal-resource [data links embedded]
  (merge data
    {:_links    (into {} (map (fn [[rel href]]
                                [rel {:href href}])
                               links))
     :_embedded embedded}))

;; Order resource with hypermedia
(defn order->hal [order base-url]
  (hal-resource
    {:id         (:id order)
     :status     (name (:status order))
     :total      (:total order)
     :created-at (str (:created-at order))}
    {:self        (str base-url "/orders/" (:id order))
     :customer    (str base-url "/customers/" (:customer-id order))
     :payment     (when (= :paid (:status order))
                    (str base-url "/payments/" (:payment-id order)))
     :cancel      (when (= :pending (:status order))
                    (str base-url "/orders/" (:id order) "/cancel"))
     :ship        (when (= :paid (:status order))
                    (str base-url "/orders/" (:id order) "/ship"))}
    {:items (map (fn [item]
                    (hal-resource item
                      {:product (str base-url "/products/" (:product-id item))}
                      {}))
                  (:items order))}))

;; JSON:API format
(defn order->json-api [order included-resources]
  {:data        {:type          "orders"
                  :id            (str (:id order))
                  :attributes    {:status     (name (:status order))
                                  :total      (:total order)
                                  :created-at (str (:created-at order))}
                  :relationships {:customer {:data {:type "customers"
                                                     :id   (str (:customer-id order))}}
                                  :items    {:data (map #(hash-map :type "order-items"
                                                                     :id (str (:id %)))
                                                         (:items order))}}}
   :included    included-resources})

;; Problem Details (RFC 7807)
(defn problem-detail [type title status detail instance]
  {:type     (str "https://api.example.com/problems/" type)
   :title    title
   :status   status
   :detail   detail
   :instance instance})

(defn not-found-problem [resource-type id]
  (problem-detail
    "not-found"
    "Resource Not Found"
    404
    (str resource-type " with id " id " was not found")
    (str "/" resource-type "/" id)))
```

---

## ขั้นตอนที่ 2642: Cursor-Based Pagination

```clojure
;; Cursor pagination: better than offset for large datasets
;; - Consistent results (no duplicate/skipped items)
;; - O(1) time with proper indexes

(defn encode-cursor [id created-at]
  (-> {:id id :ts (str created-at)}
      json/encode
      (.getBytes "UTF-8")
      java.util.Base64/getEncoder
      .encodeToString))

(defn decode-cursor [cursor-str]
  (-> cursor-str
      java.util.Base64/getDecoder
      .decode
      (String. "UTF-8")
      (json/parse-string true)))

;; List orders with cursor pagination
(defn list-orders-page [db {:keys [cursor limit filter-status]}]
  (let [limit       (min (or limit 20) 100)
        cursor-data (when cursor (decode-cursor cursor))
        
        [query params]
        (if cursor-data
          ["SELECT id, customer_id, status, total, created_at
            FROM orders
            WHERE (created_at, id) < (?, ?)
            AND (? IS NULL OR status = ?)
            ORDER BY created_at DESC, id DESC
            LIMIT ?"
           [(:ts cursor-data) (:id cursor-data)
            filter-status filter-status
            (inc limit)]]
          
          ["SELECT id, customer_id, status, total, created_at
            FROM orders
            WHERE ? IS NULL OR status = ?
            ORDER BY created_at DESC, id DESC
            LIMIT ?"
           [filter-status filter-status (inc limit)]])
        
        rows (jdbc/execute! db (into [query] params))
        
        has-next? (> (count rows) limit)
        items     (take limit rows)
        next-item (last items)]
    
    {:items    items
     :has-next has-next?
     :cursor   (when has-next?
                 (encode-cursor (:orders/id next-item)
                                (:orders/created_at next-item)))}))

;; Response format
(defn paginated-response [base-url resource-path page]
  {:data    (:items page)
   :pagination
   {:has-next-page (:has-next page)
    :next-cursor   (:cursor page)
    :next-link     (when (:cursor page)
                     (str base-url resource-path
                          "?cursor=" (:cursor page)))}})
```

---

## ขั้นตอนที่ 2643: API Versioning Strategies

```clojure
;; API versioning: URL path vs Accept header

;; Option 1: URL path versioning (/api/v1/, /api/v2/)
(defn versioned-routes []
  ["/api"
   ["/v1" {:middleware [wrap-v1-transforms]
            :get (fn [req] (v1-handler req))}]
   ["/v2" {:middleware [wrap-v2-transforms]
            :get (fn [req] (v2-handler req))}]])

;; Option 2: Accept header versioning (recommended)
;; Accept: application/vnd.myapi.v2+json
(defn extract-api-version [request]
  (let [accept (get-in request [:headers "accept"] "")]
    (or (re-find #"vnd\.myapi\.v(\d+)\+json" accept)
        "1")))

(defn wrap-api-version [handler]
  (fn [request]
    (let [version (extract-api-version request)]
      (handler (assoc request :api-version version)))))

;; Version-specific transformations
(defmulti transform-order-response
  (fn [order version] version))

(defmethod transform-order-response "1" [order _]
  ;; v1: simple format
  {:id     (:id order)
   :status (name (:status order))
   :amount (:total order)})

(defmethod transform-order-response "2" [order _]
  ;; v2: richer format with hypermedia
  (order->hal order "https://api.example.com"))

;; Deprecation warnings
(defn wrap-deprecation [handler deprecated-version sunset-date]
  (fn [request]
    (let [version (:api-version request)
          resp    (handler request)]
      (if (= version deprecated-version)
        (assoc-in resp [:headers "Deprecation"]
                   (str "true; version=" version
                        "; sunset=" sunset-date))
        resp))))
```

---

## ขั้นตอนที่ 2644: Advanced Rate Limiting

```clojure
;; Sliding window rate limiter with Redis

(defn check-rate-limit! [redis-pool key limit window-seconds]
  (let [now-ms   (System/currentTimeMillis)
        window-ms (* window-seconds 1000)
        window-start (- now-ms window-ms)]
    (car/wcar redis-pool
      ;; Remove old entries
      (car/zremrangebyscore key 0 window-start)
      ;; Count current window
      (car/zcard key)
      ;; Add current request
      (car/zadd key now-ms (str now-ms))
      ;; Set expiry
      (car/expire key (inc window-seconds)))))

(defn rate-limited? [redis-pool key limit window-seconds]
  (let [[_ count _ _] (check-rate-limit! redis-pool key limit window-seconds)]
    (> (inc count) limit)))

;; Middleware with different limits per endpoint/user
(defn wrap-rate-limit [handler redis-pool limits]
  (fn [request]
    (let [user-id      (get-in request [:user :id] "anonymous")
          endpoint     (:uri request)
          method       (name (:request-method request))
          endpoint-key (str method ":" endpoint)
          limit        (get limits endpoint-key
                           (get limits :default {:limit 100 :window 60}))]
      
      (if (rate-limited? redis-pool
                          (str "ratelimit:" user-id ":" endpoint-key)
                          (:limit limit)
                          (:window limit))
        {:status  429
         :headers {"Retry-After"       (str (:window limit))
                    "X-RateLimit-Limit" (str (:limit limit))}
         :body    {:error "Rate limit exceeded"}}
        
        (let [resp (handler request)]
          (assoc-in resp [:headers "X-RateLimit-Limit"]
                     (str (:limit limit))))))))

;; Limits config
(def api-limits
  {:default                  {:limit 100 :window 60}
   "POST:/api/auth/login"   {:limit 5   :window 300}
   "POST:/api/payments"     {:limit 10  :window 60}
   "GET:/api/search"        {:limit 30  :window 60}})
```

---

## ขั้นตอนที่ 2645: Request Validation Middleware

```clojure
;; Comprehensive request validation

(defn wrap-request-validation [handler schemas]
  (fn [request]
    (let [method      (:request-method request)
          path        (:uri request)
          schema-key  [method path]
          schema      (get schemas schema-key)]
      
      (if-not schema
        (handler request)
        
        (let [body         (:body-params request)
              params       (:path-params request)
              query        (:query-params request)
              
              body-errors  (when (:body schema)
                              (m/explain (:body schema) body))
              param-errors (when (:params schema)
                              (m/explain (:params schema) params))
              query-errors (when (:query schema)
                              (m/explain (:query schema) query))]
          
          (if (or body-errors param-errors query-errors)
            {:status 422
             :body   {:errors
                       (cond-> []
                         body-errors  (conj {:location "body"
                                              :issues   (me/humanize body-errors)})
                         param-errors (conj {:location "path"
                                              :issues   (me/humanize param-errors)})
                         query-errors (conj {:location "query"
                                              :issues   (me/humanize query-errors)}))}}
            (handler request)))))))

;; Route schemas
(def route-schemas
  {[:post "/api/orders"]
   {:body [:map
            [:customer-id :string]
            [:items [:vector
                      [:map
                        [:product-id :string]
                        [:quantity [:int {:min 1 :max 100}]]]]]]]}
   
   [:get "/api/orders"]
   {:query [:map {:closed false}
              [:status {:optional true}
               [:enum "pending" "confirmed" "paid" "shipped"]]
              [:limit {:optional true}
               [:int {:min 1 :max 100}]]]}})
```

---

*Part 89 จาก 100+ | ขั้นตอน 2641-2670 จาก 1000+*
