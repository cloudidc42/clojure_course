# Part 9: Web Development ด้วย Ring และ Reitit
## ขั้นตอนที่ 241-270: HTTP Server, Routing, Middleware

---

## บทนำ

Clojure web stack ที่นิยมในปัจจุบัน:
- **Ring** - HTTP abstraction (request/response as maps)
- **Reitit** - Fast data-driven router
- **Jetty/http-kit** - HTTP server
- **Muuntaja** - Content negotiation (JSON, EDN, Transit)

```
Request Flow:
Browser → Jetty → Ring Middleware Stack → Router → Handler → Response
```

---

## ขั้นตอนที่ 241: Ring Concept

```clojure
;; deps.edn
;; {:deps {ring/ring-core {:mvn/version "1.11.0"}
;;         ring/ring-jetty-adapter {:mvn/version "1.11.0"}
;;         metosin/reitit {:mvn/version "0.7.0"}
;;         metosin/muuntaja {:mvn/version "0.6.8"}}}

;; ===== Ring = HTTP as Data =====
;; Request = plain map
(def example-request
  {:request-method :get
   :uri "/api/users/42"
   :query-string "format=json"
   :headers {"content-type" "application/json"
              "authorization" "Bearer token123"}
   :body nil
   :params {:id "42"}})

;; Response = plain map
(def example-response
  {:status 200
   :headers {"Content-Type" "application/json"}
   :body "{\"id\": 42, \"name\": \"สมชาย\"}"})

;; Handler = function: request → response
(defn hello-handler [request]
  {:status 200
   :headers {"Content-Type" "text/plain"}
   :body "Hello, World!"})

;; Middleware = function: handler → handler
(defn wrap-logging [handler]
  (fn [request]
    (println "→" (:request-method request) (:uri request))
    (let [response (handler request)]
      (println "←" (:status response))
      response)))

;; Wrap handler with middleware
(def logged-handler (wrap-logging hello-handler))
```

---

## ขั้นตอนที่ 242: เริ่มต้น HTTP Server

```clojure
(ns myapp.server
  (:require [ring.adapter.jetty :as jetty]
            [ring.util.response :as response]))

;; Simple handler
(defn handler [request]
  (response/response "Hello from Clojure!"))

;; Start server
(defn -main []
  (jetty/run-jetty handler
    {:port 3000
     :join? false}))   ; false = non-blocking

;; Start with REPL
(def server
  (jetty/run-jetty handler
    {:port 3000
     :join? false}))

;; Stop server
(.stop server)

;; Restart
(.start server)
```

---

## ขั้นตอนที่ 243: Reitit Router

```clojure
(ns myapp.routes
  (:require [reitit.ring :as ring]
            [ring.util.response :as resp]))

;; Routes as data!
(def routes
  [["/" {:get (fn [_] (resp/response "Home"))}]
   ["/api"
    ["/users"
     ["" {:get  (fn [_] (resp/response "Get all users"))
          :post (fn [req] (resp/response "Create user"))}]
     ["/:id" {:get    (fn [req]
                        (let [id (get-in req [:path-params :id])]
                          (resp/response (str "Get user " id))))
              :put    (fn [req] (resp/response "Update user"))
              :delete (fn [req] (resp/response "Delete user"))}]]
    ["/health" {:get (fn [_] (resp/response {:status "ok"}))}]]])

;; Create router
(def router
  (ring/router routes))

;; Create ring handler
(def app
  (ring/ring-handler router
    (ring/create-default-handler)))  ; 404 handler
```

---

## ขั้นตอนที่ 244: JSON API ด้วย Muuntaja

```clojure
(ns myapp.api
  (:require [reitit.ring :as ring]
            [reitit.ring.middleware.muuntaja :as muuntaja]
            [muuntaja.core :as m]
            [ring.util.response :as resp]))

;; Return EDN/maps ตรงๆ - Muuntaja จัดการ serialization!

(defn get-users [request]
  {:status 200
   :body [{:id 1 :name "สมชาย" :email "somchai@test.com"}
          {:id 2 :name "สมหญิง" :email "somying@test.com"}]})

(defn create-user [request]
  (let [user (:body-params request)]  ; already parsed from JSON!
    {:status 201
     :body (assoc user :id (rand-int 1000))}))

(def routes
  [["/api"
    ["/users"
     ["" {:get  get-users
          :post {:handler create-user
                 :parameters {:body {:name string? :email string?}}}}]]]])

(def app
  (ring/ring-handler
    (ring/router routes
      {:data {:muuntaja m/instance
               :middleware [muuntaja/format-middleware]}})
    (ring/create-default-handler)))

;; Test:
;; curl -X POST http://localhost:3000/api/users \
;;   -H "Content-Type: application/json" \
;;   -d '{"name":"สมชาย","email":"test@test.com"}'
```

---

## ขั้นตอนที่ 245: Middleware Stack

```clojure
(ns myapp.middleware
  (:require [ring.middleware.params :refer [wrap-params]]
            [ring.middleware.keyword-params :refer [wrap-keyword-params]]
            [ring.middleware.json :refer [wrap-json-body wrap-json-response]]
            [ring.middleware.cors :refer [wrap-cors]]))

;; Custom middleware
(defn wrap-request-id [handler]
  (fn [request]
    (let [request-id (str (java.util.UUID/randomUUID))
          request (assoc request :request-id request-id)
          response (handler request)]
      (assoc-in response [:headers "X-Request-Id"] request-id))))

(defn wrap-timing [handler]
  (fn [request]
    (let [start (System/currentTimeMillis)
          response (handler request)
          elapsed (- (System/currentTimeMillis) start)]
      (assoc-in response [:headers "X-Response-Time"] (str elapsed "ms")))))

(defn wrap-auth [handler]
  (fn [request]
    (let [token (get-in request [:headers "authorization"])]
      (if (valid-token? token)
        (handler (assoc request :current-user (decode-token token)))
        {:status 401 :body "Unauthorized"}))))

;; Apply middleware stack
(defn create-app [routes]
  (-> routes
      wrap-request-id
      wrap-timing
      wrap-keyword-params
      wrap-params
      (wrap-cors :access-control-allow-origin [#".*"]
                 :access-control-allow-methods [:get :post :put :delete])))
```

---

## ขั้นตอนที่ 246: Route Parameters และ Validation

```clojure
(ns myapp.api.users
  (:require [reitit.ring :as ring]
            [reitit.coercion.malli :as rcm]
            [reitit.ring.coercion :as coercion]
            [malli.core :as m]))

;; Schema definitions
(def UserSchema
  [:map
   [:name [:string {:min 1 :max 100}]]
   [:email [:re #".+@.+\..+"]]
   [:age {:optional true} [:int {:min 0 :max 150}]]])

(def UserIdSchema
  [:map
   [:id [:int {:min 1}]]])

;; Handlers
(defn list-users [{:keys [query-params]}]
  (let [{:strs [page size]} query-params
        page (or (some-> page parse-long) 1)
        size (or (some-> size parse-long) 20)]
    {:status 200
     :body {:users (db/find-users {:page page :size size})
            :page page
            :size size}}))

(defn get-user [{{:keys [id]} :path-params}]
  (if-let [user (db/find-user-by-id id)]
    {:status 200 :body user}
    {:status 404 :body {:error "User not found"}}))

(defn create-user [{{:keys [name email age]} :body-params}]
  {:status 201
   :body (db/create-user! {:name name :email email :age age})})

;; Routes with validation
(def user-routes
  ["/users"
   ["" {:get  list-users
        :post {:handler create-user
               :parameters {:body UserSchema}}}]
   ["/:id" {:parameters {:path UserIdSchema}
            :get    get-user
            :put    {:handler update-user
                     :parameters {:body UserSchema}}
            :delete delete-user}]])
```

---

## ขั้นตอนที่ 247: Authentication และ Authorization

```clojure
(ns myapp.auth
  (:require [buddy.sign.jwt :as jwt]
            [buddy.core.keys :as keys]
            [ring.util.response :as resp]))

;; JWT configuration
(def secret "your-secret-key-at-least-256-bits")

(defn create-token [user-id roles]
  (jwt/sign {:sub user-id
             :roles roles
             :exp (+ (System/currentTimeMillis) (* 24 60 60 1000))}
            secret))

(defn verify-token [token]
  (try
    (jwt/unsign token secret)
    (catch Exception _ nil)))

;; Auth middleware
(defn wrap-authentication [handler]
  (fn [request]
    (if-let [token (some-> (get-in request [:headers "authorization"])
                            (clojure.string/replace #"^Bearer " ""))]
      (if-let [claims (verify-token token)]
        (handler (assoc request :auth claims))
        {:status 401 :body {:error "Invalid token"}})
      {:status 401 :body {:error "Missing token"}})))

;; Role-based authorization
(defn require-role [role handler]
  (fn [request]
    (let [roles (set (get-in request [:auth :roles]))]
      (if (roles role)
        (handler request)
        {:status 403 :body {:error "Forbidden"}}))))

;; Login endpoint
(defn login-handler [{{:keys [email password]} :body-params}]
  (if-let [user (auth/authenticate email password)]
    {:status 200
     :body {:token (create-token (:id user) (:roles user))
            :user user}}
    {:status 401
     :body {:error "Invalid credentials"}}))
```

---

## ขั้นตอนที่ 248: Error Handling Middleware

```clojure
(ns myapp.middleware.errors
  (:require [ring.util.response :as resp]
            [taoensso.timbre :as log]))

;; Global exception handler
(defn wrap-exception-handler [handler]
  (fn [request]
    (try
      (handler request)
      (catch clojure.lang.ExceptionInfo e
        (let [{:keys [status message data]} (ex-data e)]
          (log/warn "Application error:" message data)
          {:status (or status 400)
           :body {:error message
                  :details data}}))
      (catch Exception e
        (log/error e "Unexpected error handling request"
                   {:uri (:uri request)
                    :method (:request-method request)})
        {:status 500
         :body {:error "Internal server error"}}))))

;; Domain-specific exceptions
(defn not-found! [resource id]
  (throw (ex-info (str resource " not found")
                  {:status 404
                   :message (str resource " with id " id " not found")})))

(defn validation-error! [errors]
  (throw (ex-info "Validation failed"
                  {:status 422
                   :message "Validation failed"
                   :data {:errors errors}})))

(defn unauthorized! []
  (throw (ex-info "Unauthorized"
                  {:status 401
                   :message "Authentication required"})))

;; ใช้งาน
(defn get-user [{{:keys [id]} :path-params}]
  (or (db/find-user id)
      (not-found! "User" id)))
```

---

## ขั้นตอนที่ 249: Database Integration

```clojure
(ns myapp.db
  (:require [next.jdbc :as jdbc]
            [next.jdbc.sql :as sql]
            [honey.sql :as honey]))

;; Database connection
(def db
  (jdbc/get-datasource
    {:dbtype "postgresql"
     :dbname "myapp"
     :host "localhost"
     :port 5432
     :user "postgres"
     :password "password"}))

;; Simple queries
(defn find-all-users []
  (sql/find-by-keys db :users :all))

(defn find-user-by-id [id]
  (sql/get-by-id db :users id))

(defn create-user! [user]
  (sql/insert! db :users user))

(defn update-user! [id user]
  (sql/update! db :users user {:id id}))

(defn delete-user! [id]
  (sql/delete! db :users {:id id}))

;; HoneySQL for complex queries
(defn search-users [{:keys [name email page size]}]
  (let [page (or page 1)
        size (or size 20)
        query (cond-> {:select [:*]
                       :from [:users]
                       :order-by [:name]
                       :limit size
                       :offset (* (dec page) size)}
                name  (assoc :where [:ilike :name (str "%" name "%")])
                email (assoc :where [:= :email email]))]
    (jdbc/execute! db (honey/format query))))

;; Transaction
(defn transfer-credits! [from-id to-id amount]
  (jdbc/with-transaction [tx db]
    (let [from (sql/get-by-id tx :accounts from-id)
          to   (sql/get-by-id tx :accounts to-id)]
      (when (< (:balance from) amount)
        (throw (ex-info "Insufficient balance" {:status 422})))
      (sql/update! tx :accounts {:balance (- (:balance from) amount)} {:id from-id})
      (sql/update! tx :accounts {:balance (+ (:balance to) amount)} {:id to-id})
      {:success true})))
```

---

## ขั้นตอนที่ 250: Complete REST API

```clojure
(ns myapp.core
  (:require [reitit.ring :as ring]
            [reitit.ring.middleware.muuntaja :as muuntaja]
            [reitit.ring.coercion :as coercion]
            [muuntaja.core :as m]
            [ring.adapter.jetty :as jetty]
            [myapp.api.users :as users]
            [myapp.api.auth :as auth]
            [myapp.middleware :as mw]))

(def api-routes
  ["/api/v1"
   ["/auth"
    ["/login"  {:post auth/login}]
    ["/logout" {:post auth/logout}]]
   ["/users" {:middleware [mw/wrap-authentication]}
    ["" {:get  users/list-users
         :post users/create-user}]
    ["/:id" {:get    users/get-user
             :put    users/update-user
             :delete users/delete-user}]]
   ["/health" {:get (fn [_] {:status 200 :body {:status "ok"}})}]])

(def app
  (ring/ring-handler
    (ring/router
      api-routes
      {:data {:muuntaja m/instance
               :middleware [muuntaja/format-middleware
                            coercion/coerce-request-middleware
                            coercion/coerce-response-middleware]}})
    (ring/create-default-handler
      {:not-found (constantly {:status 404 :body {:error "Not found"}})
       :method-not-allowed (constantly {:status 405 :body {:error "Method not allowed"}})})))

(defn -main [& args]
  (println "Starting server on port 3000...")
  (jetty/run-jetty
    (-> app
        mw/wrap-exception-handler
        mw/wrap-request-id
        mw/wrap-timing)
    {:port 3000 :join? true}))
```

---

## ขั้นตอนที่ 251: Rate Limiting Middleware

```clojure
(ns myapp.middleware.rate-limit
  (:require [clojure.core.cache :as cache]))

;; Token bucket rate limiter
(defn make-rate-limiter [max-requests window-ms]
  (atom (cache/ttl-cache-factory {} :ttl window-ms)))

(defn check-rate-limit! [limiter key]
  (let [count (get @limiter key 0)]
    (if (>= count max-requests)
      false
      (do (swap! limiter update key (fnil inc 0))
          true))))

(def api-limiter (make-rate-limiter 100 60000))  ; 100 req/min

(defn wrap-rate-limit [handler]
  (fn [request]
    (let [client-ip (or (get-in request [:headers "x-forwarded-for"])
                         (:remote-addr request))]
      (if (check-rate-limit! api-limiter client-ip)
        (handler request)
        {:status 429
         :headers {"Retry-After" "60"}
         :body {:error "Too many requests"}}))))
```

---

## Project Exercise: Blog API

```clojure
(ns blog-api.core
  (:require [reitit.ring :as ring]
            [muuntaja.core :as m]
            [reitit.ring.middleware.muuntaja :as muuntaja]
            [ring.adapter.jetty :as jetty]
            [next.jdbc :as jdbc]))

;; In-memory database (ใช้ atom แทน PostgreSQL ตอนทดสอบ)
(def db (atom {:posts {}
               :comments {}
               :next-id 1}))

(defn next-id! []
  (:next-id (swap! db update :next-id inc)))

;; Posts handlers
(defn list-posts [_]
  {:status 200
   :body (vals (:posts @db))})

(defn get-post [{{:keys [id]} :path-params}]
  (if-let [post (get-in @db [:posts (parse-long id)])]
    {:status 200 :body post}
    {:status 404 :body {:error "Post not found"}}))

(defn create-post [{{:keys [title content author]} :body-params}]
  (let [id (next-id!)
        post {:id id :title title :content content :author author
              :created-at (str (java.time.Instant/now))
              :comments []}]
    (swap! db assoc-in [:posts id] post)
    {:status 201 :body post}))

(defn delete-post [{{:keys [id]} :path-params}]
  (swap! db update :posts dissoc (parse-long id))
  {:status 204})

;; Comments handlers
(defn add-comment [{{:keys [id]} :path-params
                    {:keys [author content]} :body-params}]
  (let [post-id (parse-long id)]
    (if (get-in @db [:posts post-id])
      (let [comment {:id (next-id!) :author author :content content
                     :created-at (str (java.time.Instant/now))}]
        (swap! db update-in [:posts post-id :comments] conj comment)
        {:status 201 :body comment})
      {:status 404 :body {:error "Post not found"}})))

;; Routes
(def routes
  ["/api"
   ["/posts"
    ["" {:get  list-posts
         :post create-post}]
    ["/:id"
     ["" {:get    get-post
          :delete delete-post}]
     ["/comments" {:post add-comment}]]]])

(def app
  (ring/ring-handler
    (ring/router routes
      {:data {:muuntaja m/instance
               :middleware [muuntaja/format-middleware]}})
    (ring/create-default-handler)))

;; REPL testing
(comment
  (def server (jetty/run-jetty app {:port 3000 :join? false}))
  
  ;; Test
  ;; POST /api/posts
  ;; GET  /api/posts
  ;; GET  /api/posts/1
  ;; POST /api/posts/1/comments
  ;; DELETE /api/posts/1
  )
```

---

### สรุป Web Stack

```
Ring:
  - Request/Response เป็น plain maps
  - Handlers = functions
  - Middleware = function wrappers

Reitit:
  - Routes เป็น data (vectors)
  - Fast routing (compiled)
  - Built-in coercion/validation

Muuntaja:
  - Content negotiation
  - Auto JSON/EDN/Transit serialization

Pattern:
  Request → Middleware → Router → Handler → Response
```

---

*Part 9 จาก 100+ | ขั้นตอน 241-270 จาก 1000+*
