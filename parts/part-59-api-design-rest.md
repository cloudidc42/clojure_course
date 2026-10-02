# Part 59: API Design RESTful ขั้นสูง
## ขั้นตอนที่ 1741-1770: OpenAPI, Versioning, Hypermedia, Rate Limiting, API Keys

---

## บทนำ

API Design ระดับ production:
- **OpenAPI/Swagger** - API documentation
- **Versioning** - backward compatibility
- **Hypermedia (HATEOAS)** - self-describing APIs
- **API keys** - authentication
- **Idempotency** - safe retries

---

## ขั้นตอนที่ 1741: OpenAPI Specification

```clojure
(ns myapp.api.spec
  (:require [reitit.swagger :as swagger]
            [reitit.swagger-ui :as swagger-ui]
            [reitit.coercion.malli :as malli-coercion]
            [malli.core :as m]))

;; Schemas
(def ProductSchema
  [:map
   [:id        {:description "Product UUID"} uuid?]
   [:name      {:description "Product name"} string?]
   [:price     {:description "Price in USD"} number?]
   [:sku       {:description "Stock keeping unit"} string?]
   [:stock     {:description "Available stock"} int?]
   [:category  {:description "Product category"} string?]
   [:active?   {:description "Is product active"} boolean?]])

(def CreateProductSchema
  [:map
   [:name     string?]
   [:price    pos?]
   [:sku      string?]
   [:category string?]])

;; Router with OpenAPI
(def router
  (reitit/router
    ["/api"
     {:swagger {:info {:title       "MyApp API"
                        :description "E-Commerce API"
                        :version     "1.0.0"
                        :contact     {:name  "API Team"
                                       :email "api@myapp.com"}}
                 :securityDefinitions
                 {:api-key {:type "apiKey" :in "header" :name "X-API-Key"}
                  :bearer  {:type "http" :scheme "bearer" :bearerFormat "JWT"}}}}

     ["/swagger.json"
      {:get {:no-doc true
              :handler (swagger/create-swagger-handler)}}]

     ["/products"
      {:get  {:summary     "List products"
               :description "Returns paginated list of products"
               :tags        ["Products"]
               :parameters  {:query [:map
                                       [:page  {:optional true} pos-int?]
                                       [:limit {:optional true} pos-int?]
                                       [:q     {:optional true} string?]]}
               :responses   {200 {:body [:map
                                           [:items [:vector ProductSchema]]
                                           [:total int?]
                                           [:page  int?]]}}
               :handler list-products-handler}

       :post {:summary    "Create product"
               :tags       ["Products"]
               :security   [{:api-key []}]
               :parameters {:body CreateProductSchema}
               :responses  {201 {:body ProductSchema}
                             422 {:body [:map [:errors [:vector string?]]]}}
               :handler    create-product-handler}}]

     ["/products/:id"
      {:get    {:summary    "Get product"
                 :tags       ["Products"]
                 :parameters {:path [:map [:id uuid?]]}
                 :responses  {200 {:body ProductSchema}
                               404 {:body [:map [:error string?]]}}}
       :put    {:summary    "Update product"
                 :tags       ["Products"]
                 :security   [{:api-key []}]
                 :parameters {:path [:map [:id uuid?]]
                               :body CreateProductSchema}
                 :responses  {200 {:body ProductSchema}}}
       :delete {:summary    "Delete product"
                 :tags       ["Products"]
                 :security   [{:api-key []}]
                 :parameters {:path [:map [:id uuid?]]}
                 :responses  {204 {:body nil}}}}]]))
```

---

## ขั้นตอนที่ 1742: API Versioning

```clojure
(ns myapp.api.versioning
  (:require [reitit.ring :as ring]))

;; Strategy 1: URL versioning (simplest)
;; /api/v1/products, /api/v2/products

(defn v1-product-response [product]
  {:id    (:id product)
   :name  (:name product)
   :price (:price product)})

(defn v2-product-response [product]
  {:id          (:id product)
   :name        (:name product)
   :price       (:price product)
   :sku         (:sku product)
   :stock       (:stock product)
   :category    (:category product)
   :_links      {:self   {:href (str "/api/v2/products/" (:id product))}
                  :update {:href (str "/api/v2/products/" (:id product))
                           :method "PUT"}}})

(def api-v1-router
  (ring/router
    ["/api/v1"
     ["/products" {:get list-products-v1}]
     ["/products/:id" {:get get-product-v1}]]))

(def api-v2-router
  (ring/router
    ["/api/v2"
     ["/products" {:get list-products-v2}]
     ["/products/:id" {:get get-product-v2}]]))

;; Strategy 2: Header versioning
;; Accept: application/vnd.myapp.v2+json

(defn version-middleware [handler]
  (fn [request]
    (let [accept  (get-in request [:headers "accept"] "")
          version (cond
                    (clojure.string/includes? accept "v2") :v2
                    (clojure.string/includes? accept "v1") :v1
                    :else :v2)]  ; default to latest
      (handler (assoc request :api-version version)))))

;; Strategy 3: Deprecation headers
(defn deprecation-middleware [deprecated-at sunset-at]
  (fn [handler]
    (fn [request]
      (let [response (handler request)]
        (assoc-in response [:headers "Deprecation"]
                   (str "date=\"" deprecated-at "\""))
        (assoc-in response [:headers "Sunset"]
                   sunset-at)))))
```

---

## ขั้นตอนที่ 1743: Hypermedia (HATEOAS)

```clojure
(ns myapp.api.hypermedia)

;; HATEOAS: responses include links to related resources
;; Makes API self-describing and navigable

(defn link [rel href & {:keys [method title]}]
  (cond-> {:href href}
    rel    (assoc :rel rel)
    method (assoc :method method)
    title  (assoc :title title)))

(defn product-links [product base-url]
  {:self     (link :self (str base-url "/products/" (:id product)))
   :update   (link :update (str base-url "/products/" (:id product))
                    :method "PUT" :title "Update product")
   :delete   (link :delete (str base-url "/products/" (:id product))
                    :method "DELETE" :title "Delete product")
   :category (link :category (str base-url "/categories/" (:category-id product))
                    :title "Product category")})

(defn embed-links [resource links]
  (assoc resource :_links links))

;; Collection response with links
(defn product-collection-response [products page total base-url]
  {:_embedded {:products (map #(embed-links % (product-links % base-url)) products)}
   :_links    {:self  (link :self  (str base-url "/products?page=" page))
                :next  (when (< (* page 20) total)
                         (link :next (str base-url "/products?page=" (inc page))))
                :prev  (when (> page 1)
                         (link :prev (str base-url "/products?page=" (dec page))))
                :first (link :first (str base-url "/products?page=1"))
                :last  (link :last  (str base-url "/products?page=" (Math/ceil (/ total 20))))}
   :_metadata {:total   total
                :page    page
                :pages   (Math/ceil (/ total 20))}})
```

---

## ขั้นตอนที่ 1744: API Keys Authentication

```clojure
(ns myapp.api.auth
  (:require [next.jdbc :as jdbc]
            [buddy.core.codecs :as codecs]
            [buddy.core.hash :as hash]))

;; API key storage
;; CREATE TABLE api_keys (
;;   id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
;;   key_hash   VARCHAR(64) NOT NULL UNIQUE,
;;   name       VARCHAR(100) NOT NULL,
;;   user_id    UUID REFERENCES users(id),
;;   scopes     TEXT[] DEFAULT '{}',
;;   rate_limit INT DEFAULT 1000,
;;   active     BOOLEAN DEFAULT TRUE,
;;   created_at TIMESTAMPTZ DEFAULT NOW(),
;;   last_used  TIMESTAMPTZ
;; );

(defn generate-api-key []
  (let [random-bytes (byte-array 32)]
    (.nextBytes (java.security.SecureRandom.) random-bytes)
    (str "sk_" (codecs/bytes->hex random-bytes))))

(defn hash-key [api-key]
  (codecs/bytes->hex (hash/sha256 api-key)))

(defn create-api-key! [db user-id name scopes]
  (let [key      (generate-api-key)
        key-hash (hash-key key)]
    (jdbc/execute-one! db
      ["INSERT INTO api_keys (key_hash, name, user_id, scopes)
        VALUES (?, ?, ?, ?)
        RETURNING id, name, created_at"
       key-hash name user-id (into-array String scopes)])
    {:key key}))  ; Return plain key ONCE, store only hash

(defn verify-api-key [db api-key]
  (let [key-hash (hash-key api-key)]
    (when-let [record (jdbc/execute-one! db
                        ["SELECT ak.*, u.id as user_id, u.email
                          FROM api_keys ak
                          JOIN users u ON ak.user_id = u.id
                          WHERE ak.key_hash = ?
                          AND ak.active = TRUE"
                         key-hash])]
      (jdbc/execute-one! db
        ["UPDATE api_keys SET last_used = NOW() WHERE key_hash = ?"
         key-hash])
      record)))

;; Middleware
(defn wrap-api-key-auth [handler db]
  (fn [request]
    (let [api-key (or (get-in request [:headers "x-api-key"])
                       (get-in request [:query-params "api_key"]))]
      (if api-key
        (if-let [key-record (verify-api-key db api-key)]
          (handler (assoc request
                           :auth-key  key-record
                           :user-id   (:api_keys/user_id key-record)
                           :api-scopes (set (:api_keys/scopes key-record))))
          {:status 401 :body {:error "Invalid API key"}})
        {:status 401 :body {:error "Missing API key"}}))))

;; Scope check
(defn require-scope [scope]
  (fn [handler]
    (fn [request]
      (if (contains? (:api-scopes request) scope)
        (handler request)
        {:status 403 :body {:error (str "Requires scope: " scope)}}))))
```

---

## ขั้นตอนที่ 1745: Idempotency Keys

```clojure
(ns myapp.api.idempotency
  (:require [next.jdbc :as jdbc]))

;; Idempotency: same request = same result
;; Client sends Idempotency-Key header
;; Server stores result, returns same on retry

(defn get-cached-response [db idempotency-key]
  (jdbc/execute-one! db
    ["SELECT response_status, response_body, created_at
      FROM idempotency_cache
      WHERE key = ?
      AND created_at > NOW() - INTERVAL '24 hours'"
     idempotency-key]))

(defn cache-response! [db idempotency-key status body]
  (jdbc/execute-one! db
    ["INSERT INTO idempotency_cache (key, response_status, response_body)
      VALUES (?, ?, ?::jsonb)
      ON CONFLICT (key) DO NOTHING"
     idempotency-key status (cheshire.core/generate-string body)]))

(defn wrap-idempotency [handler db]
  (fn [request]
    (if-let [idem-key (get-in request [:headers "idempotency-key"])]
      (if-let [cached (get-cached-response db idem-key)]
        ;; Return cached response
        {:status  (:idempotency_cache/response_status cached)
         :headers {"Idempotency-Key"     idem-key
                    "X-Idempotency-Cache" "hit"}
         :body    (cheshire.core/parse-string
                    (:idempotency_cache/response_body cached) true)}
        
        ;; Process and cache
        (let [response (handler request)]
          (when (< (:status response) 500)
            (cache-response! db idem-key (:status response) (:body response)))
          (assoc-in response [:headers "Idempotency-Key"] idem-key)))
      
      ;; No idempotency key - process normally
      (handler request))))
```

---

## Project: Complete RESTful API

```clojure
(ns myapp.api.complete
  (:require [reitit.ring :as ring]
            [reitit.coercion.malli :as malli]))

;; Complete production API with all features
(def app
  (-> (ring/ring-handler
        (ring/router routes
          {:data {:coercion   malli/coercion
                  :middleware [swagger/swagger-feature
                               parameters/parameters-middleware
                               muuntaja/format-middleware
                               coercion/coerce-exceptions-middleware
                               coercion/coerce-request-middleware
                               coercion/coerce-response-middleware]}})
        (ring/routes
          (swagger-ui/create-swagger-ui-handler {:path "/api/docs"})
          (ring/create-default-handler)))
      
      ;; Middleware stack
      (wrap-idempotency db)
      (wrap-api-key-auth db)
      (wrap-rate-limit redis)
      (wrap-request-id)
      (wrap-cors {:origins    #{"https://myapp.com"}
                   :methods    [:get :post :put :delete]
                   :headers    ["Content-Type" "X-API-Key"]})
      (wrap-json-body {:keywords? true})
      (wrap-error-handler)
      (wrap-access-log)))
```

---

*Part 59 จาก 100+ | ขั้นตอน 1741-1770 จาก 1000+*
