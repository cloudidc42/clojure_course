# Part 31: Pedestal Framework
## ขั้นตอนที่ 901-930: Interceptors, Routes, SSE, Service Maps

---

## บทนำ

**Pedestal** เป็น Clojure web framework ที่:
- ใช้ **Interceptors** แทน middleware (composable, bidirectional)
- รองรับ **async** โดย native
- มี **routing** ที่ powerful
- รองรับ **HTTP/2** และ **SSE**
- เหมาะสำหรับ high-performance APIs

---

## ขั้นตอนที่ 901: Pedestal Setup

```clojure
;; deps.edn
;; {:deps {io.pedestal/pedestal.service    {:mvn/version "0.6.3"}
;;         io.pedestal/pedestal.jetty      {:mvn/version "0.6.3"}
;;         io.pedestal/pedestal.route      {:mvn/version "0.6.3"}
;;         org.slf4j/slf4j-simple          {:mvn/version "2.0.9"}}}

(ns myapp.service
  (:require [io.pedestal.http :as http]
            [io.pedestal.http.route :as route]))

;; Simple handler = function: request → response
(defn hello-handler [request]
  {:status  200
   :headers {"Content-Type" "application/json"}
   :body    "{\"message\": \"Hello, Pedestal!\"}"})

;; Routes table
(def routes
  #{["/hello" :get hello-handler :route-name :hello]
    ["/api/users" :get #'get-users-handler :route-name :get-users]
    ["/api/users/:id" :get #'get-user-handler :route-name :get-user]
    ["/api/users" :post #'create-user-handler :route-name :create-user]})

;; Service map
(def service-map
  {::http/routes routes
   ::http/type   :jetty
   ::http/port   8080
   ::http/join?  false})

;; Start server
(defonce server (atom nil))

(defn start! []
  (reset! server (-> service-map
                     http/create-server
                     http/start)))

(defn stop! []
  (when @server
    (http/stop @server)
    (reset! server nil)))
```

---

## ขั้นตอนที่ 902: Interceptors

```clojure
;; Interceptor = map ที่มี :enter, :leave, :error functions
;; ต่างจาก middleware ตรง: bidirectional chain

(ns myapp.interceptors
  (:require [io.pedestal.interceptor :as interceptor]
            [io.pedestal.http.body-params :as body-params]))

;; Simple interceptor
(def log-interceptor
  (interceptor/interceptor
    {:name  ::log
     :enter (fn [ctx]
               (let [req (:request ctx)]
                 (println "→ " (:request-method req) (:uri req)))
               ctx)
     :leave (fn [ctx]
               (let [resp (:response ctx)]
                 (println "← " (:status resp)))
               ctx)}))

;; Auth interceptor
(def auth-interceptor
  {:name  ::auth
   :enter (fn [ctx]
              (let [token (get-in ctx [:request :headers "authorization"])]
                (if-let [claims (verify-jwt token)]
                  (assoc-in ctx [:request :user] claims)
                  ;; Short-circuit: set response immediately
                  (assoc ctx :response
                         {:status 401
                          :body   "{\"error\": \"Unauthorized\"}"}))))})

;; Error handling interceptor
(def error-handler
  {:name  ::error-handler
   :error (fn [ctx error]
             (assoc ctx :response
               {:status 500
                :body   (str "{\"error\": \"" (.getMessage error) "\"}")}))})

;; Body coercion interceptor
(def json-body
  (body-params/body-params
    (body-params/default-parser-map
      :json-options {:key-fn keyword})))
```

---

## ขั้นตอนที่ 903: Routing ขั้นสูง

```clojure
;; Pedestal routing: terse table format

(def routes
  (route/expand-routes
    #{;; Simple routes
      ["/health" :get health-handler :route-name :health]
      
      ;; With interceptors
      ["/api/users"     :get  [auth-interceptor get-users-handler]  :route-name :users/list]
      ["/api/users"     :post [auth-interceptor json-body create-user-handler] :route-name :users/create]
      ["/api/users/:id" :get  [auth-interceptor get-user-handler]   :route-name :users/get]
      ["/api/users/:id" :put  [auth-interceptor json-body update-user-handler] :route-name :users/update]
      
      ;; Path parameters
      ["/api/posts/:post-id/comments" :get get-comments-handler]
      
      ;; Query string (auto-parsed)
      ["/api/search" :get search-handler]}))

;; Hierarchical routes with shared interceptors
(def api-routes
  (route/expand-routes
    #{^{:interceptors [log-interceptor error-handler]}
      ["/api"
       ^{:interceptors [auth-interceptor json-body]}
       ["/users"
        ["" :get get-users-handler :route-name :users/list]
        ["" :post create-user-handler :route-name :users/create]
        ["/:id"
         ["" :get get-user-handler :route-name :users/get]]]
       
       ["/products"
        ["" :get get-products-handler]
        ["/:id" :get get-product-handler]]]}))
```

---

## ขั้นตอนที่ 904: Context Threading

```clojure
;; Request context = map ที่ไหลผ่าน interceptor chain

(defn get-user-handler [request]
  (let [id         (get-in request [:path-params :id])
        user       (db/find-user id)
        ;; current user added by auth interceptor
        current-user (:user request)]
    
    (if user
      {:status  200
       :headers {"Content-Type" "application/json"}
       :body    (cheshire.core/generate-string user)}
      {:status  404
       :body    "{\"error\": \"Not found\"}"})))

;; Async handler using channel
(defn async-handler [request]
  (let [respond (:async-channel request)]  ; Pedestal async
    (future
      (let [result (long-running-operation (:params request))]
        (io.pedestal.http.impl.servlet-interceptor/write-response
          respond
          {:status 200
           :body (cheshire.core/generate-string result)})))
    ;; Return async marker
    :io.pedestal.http/async))

;; Content negotiation
(require '[io.pedestal.http.content-negotiation :as conneg])

(def content-neg-interceptor
  (conneg/negotiate-content
    ["application/json" "application/edn" "text/html"]))
```

---

## ขั้นตอนที่ 905: SSE ด้วย Pedestal

```clojure
(ns myapp.sse
  (:require [io.pedestal.http.sse :as sse]
            [clojure.core.async :as async]))

;; SSE handler
(defn sse-handler [event-ch ctx]
  (let [send-event! (fn [event-type data]
                       (async/>!! event-ch
                         {:name event-type
                          :data (cheshire.core/generate-string data)}))]
    
    ;; Subscribe to events
    (let [sub-ch (subscribe-to-user-events (get-in ctx [:request :user :id]))]
      
      ;; Forward events
      (async/go-loop []
        (when-let [event (async/<! sub-ch)]
          (send-event! (:type event) (:data event))
          (recur)))
      
      ;; Return cleanup function
      (fn []
        (async/close! sub-ch)))))

;; Register SSE route
(def routes
  #{["/events" :get (sse/start-event-stream sse-handler)
     :route-name :sse]})
```

---

## ขั้นตอนที่ 906: Testing Pedestal Services

```clojure
(ns myapp.service-test
  (:require [clojure.test :refer :all]
            [io.pedestal.test :as pt]
            [myapp.service :as service]))

;; Test service (without starting real server)
(def test-service
  (-> service/service-map
      http/create-servlet))

(deftest test-hello
  (let [response (pt/response-for test-service :get "/hello")]
    (is (= 200 (:status response)))
    (is (= "application/json"
           (get-in response [:headers "Content-Type"])))))

(deftest test-create-user
  (let [response (pt/response-for test-service
                   :post "/api/users"
                   :headers {"Content-Type" "application/json"
                              "Authorization" (str "Bearer " (create-test-token))}
                   :body (cheshire.core/generate-string
                           {:name  "Test User"
                            :email "test@example.com"}))]
    (is (= 201 (:status response)))
    (let [body (cheshire.core/parse-string (:body response) true)]
      (is (= "Test User" (:name body))))))

;; Test with authentication
(deftest test-unauthorized
  (let [response (pt/response-for test-service
                   :get "/api/users"
                   :headers {})]
    (is (= 401 (:status response)))))
```

---

## Project: Complete REST API ด้วย Pedestal

```clojure
(ns api.service
  (:require [io.pedestal.http :as http]
            [io.pedestal.http.route :as route]
            [io.pedestal.interceptor :as interceptor]))

;; Interceptor stack
(def common-interceptors
  [log-interceptor
   error-handler
   content-neg-interceptor])

(def api-interceptors
  (conj common-interceptors auth-interceptor json-body))

;; Handlers
(defn list-products [request]
  (let [page  (Integer/parseInt (get-in request [:query-params :page] "1"))
        limit (Integer/parseInt (get-in request [:query-params :limit] "20"))
        query (get-in request [:query-params :q] "")
        data  (db/list-products {:page page :limit limit :query query})]
    {:status 200
     :body   data}))

(defn get-product [request]
  (let [id      (get-in request [:path-params :id])
        product (db/get-product id)]
    (if product
      {:status 200 :body product}
      {:status 404 :body {:error "Product not found"}})))

;; Full routes
(def routes
  (route/expand-routes
    #{["/health" :get (conj common-interceptors health-handler) :route-name :health]
      
      ["/api/v1/products"     :get  (conj api-interceptors list-products) :route-name :products/list]
      ["/api/v1/products/:id" :get  (conj api-interceptors get-product)   :route-name :products/get]
      ["/api/v1/products"     :post (conj api-interceptors create-product) :route-name :products/create]
      
      ["/api/v1/orders"       :get  (conj api-interceptors list-orders)   :route-name :orders/list]
      ["/api/v1/orders"       :post (conj api-interceptors create-order)  :route-name :orders/create]}))

(def service
  {::http/routes  routes
   ::http/type    :jetty
   ::http/port    8080
   ::http/host    "0.0.0.0"
   ::http/join?   false
   ::http/allowed-origins ["http://localhost:3000"]})
```

---

*Part 31 จาก 100+ | ขั้นตอน 901-930 จาก 1000+*
