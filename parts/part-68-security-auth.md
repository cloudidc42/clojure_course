# Part 68: Security และ Authentication
## ขั้นตอนที่ 2011-2040: OAuth2, JWT, RBAC, Security Headers, Input Validation

---

## บทนำ

Security ใน Clojure web applications:
- **OAuth2/OIDC** - delegated authorization
- **JWT** - stateless tokens
- **RBAC** - role-based access control
- **Security headers** - protect browsers
- **Input validation** - prevent injection

---

## ขั้นตอนที่ 2011: JWT Authentication

```clojure
(ns myapp.auth.jwt
  (:require [buddy.sign.jwt :as jwt]
            [buddy.core.keys :as keys]
            [clojure.java.io :as io]))

;; Load RSA keys for asymmetric JWT (RS256)
(def private-key
  (keys/private-key (io/resource "keys/private.pem")))

(def public-key
  (keys/public-key (io/resource "keys/public.pem")))

;; Create JWT token
(defn create-access-token [user-id roles]
  (let [now     (java.time.Instant/now)
        expires (.plusSeconds now 3600)]  ; 1 hour
    (jwt/sign
      {:sub     (str user-id)
       :iat     (.getEpochSecond now)
       :exp     (.getEpochSecond expires)
       :roles   roles
       :jti     (str (java.util.UUID/randomUUID))}  ; JWT ID for revocation
      private-key
      {:alg :rs256})))

(defn create-refresh-token [user-id]
  (let [now     (java.time.Instant/now)
        expires (.plusSeconds now (* 30 24 3600))]  ; 30 days
    (jwt/sign
      {:sub  (str user-id)
       :iat  (.getEpochSecond now)
       :exp  (.getEpochSecond expires)
       :type "refresh"
       :jti  (str (java.util.UUID/randomUUID))}
      private-key
      {:alg :rs256})))

;; Verify and decode JWT
(defn verify-token [token]
  (try
    {:ok (jwt/unsign token public-key {:alg :rs256})}
    (catch clojure.lang.ExceptionInfo e
      {:error (ex-message e)})
    (catch Exception e
      {:error "Invalid token"})))

;; Middleware: extract and verify JWT
(defn wrap-jwt-auth [handler]
  (fn [request]
    (let [auth-header (get-in request [:headers "authorization"])
          token       (when auth-header
                        (second (re-matches #"Bearer (.+)" auth-header)))]
      (if-not token
        {:status 401 :body {:error "No token provided"}}
        (let [result (verify-token token)]
          (if (:error result)
            {:status 401 :body {:error (:error result)}}
            (handler (assoc request :user (:ok result)))))))))
```

---

## ขั้นตอนที่ 2012: Role-Based Access Control

```clojure
(ns myapp.auth.rbac)

;; Permission definitions
(def permissions
  {:products/read   #{:admin :editor :viewer}
   :products/write  #{:admin :editor}
   :products/delete #{:admin}
   :orders/read     #{:admin :editor :viewer}
   :orders/write    #{:admin}
   :users/read      #{:admin}
   :users/write     #{:admin}})

;; Hierarchical roles
(def role-hierarchy
  {:admin   #{:editor}
   :editor  #{:viewer}
   :viewer  #{}})

(defn get-all-roles [role]
  (loop [roles #{role}
         to-expand #{role}]
    (if (empty? to-expand)
      roles
      (let [expanded  (mapcat #(get role-hierarchy % #{}) to-expand)
            new-roles (remove roles expanded)]
        (recur (into roles expanded)
               (set new-roles))))))

(defn has-permission? [user-roles permission]
  (let [all-user-roles (set (mapcat get-all-roles user-roles))
        allowed-roles  (get permissions permission #{})]
    (some all-user-roles allowed-roles)))

;; Authorization middleware
(defn require-permission [permission]
  (fn [handler]
    (fn [request]
      (let [user-roles (get-in request [:user :roles] [])]
        (if (has-permission? (map keyword user-roles) permission)
          (handler request)
          {:status 403
           :body   {:error "Insufficient permissions"
                    :required (name permission)}})))))

;; Usage in routes
(def routes
  [["/api/products"
    {:get    {:handler list-products-handler}
     :post   {:middleware [(require-permission :products/write)]
              :handler    create-product-handler}}]
   ["/api/products/:id"
    {:delete {:middleware [(require-permission :products/delete)]
              :handler    delete-product-handler}}]])
```

---

## ขั้นตอนที่ 2013: OAuth2 Integration

```clojure
(ns myapp.auth.oauth2
  (:require [hato.client :as http]
            [ring.util.response :as response]))

(def google-config
  {:client-id     (System/getenv "GOOGLE_CLIENT_ID")
   :client-secret (System/getenv "GOOGLE_CLIENT_SECRET")
   :redirect-uri  "https://myapp.com/auth/google/callback"
   :auth-uri      "https://accounts.google.com/o/oauth2/v2/auth"
   :token-uri     "https://oauth2.googleapis.com/token"
   :userinfo-uri  "https://www.googleapis.com/oauth2/v2/userinfo"
   :scopes        ["openid" "email" "profile"]})

;; Step 1: Redirect to OAuth provider
(defn oauth2-authorize [config state]
  (let [params {:client_id     (:client-id config)
                :redirect_uri  (:redirect-uri config)
                :response_type "code"
                :scope         (clojure.string/join " " (:scopes config))
                :state         state
                :access_type   "offline"  ; request refresh token
                :prompt        "consent"}
        query-string (clojure.string/join "&"
                       (map (fn [[k v]] (str (name k) "=" (java.net.URLEncoder/encode (str v) "UTF-8")))
                            params))]
    (response/redirect (str (:auth-uri config) "?" query-string))))

;; Step 2: Handle callback and exchange code for tokens
(defn oauth2-callback [config code]
  (let [response (http/post (:token-uri config)
                   {:form-params {:code          code
                                   :client_id     (:client-id config)
                                   :client_secret (:client-secret config)
                                   :redirect_uri  (:redirect-uri config)
                                   :grant_type    "authorization_code"}
                    :as :json})]
    (:body response)))

;; Step 3: Get user info
(defn get-user-info [config access-token]
  (let [response (http/get (:userinfo-uri config)
                   {:headers {"Authorization" (str "Bearer " access-token)}
                    :as      :json})]
    (:body response)))

;; OAuth2 callback handler
(defn google-callback-handler [db request]
  (let [code  (get-in request [:params :code])
        state (get-in request [:params :state])]
    ;; Verify state to prevent CSRF
    (when-not (verify-oauth-state! state)
      (throw (ex-info "Invalid OAuth state" {:status 400})))
    
    (let [tokens    (oauth2-callback google-config code)
          user-info (get-user-info google-config (:access_token tokens))
          user      (upsert-oauth-user! db
                      {:email        (:email user-info)
                       :name         (:name user-info)
                       :picture      (:picture user-info)
                       :provider     "google"
                       :provider-id  (:sub user-info)
                       :refresh-token (:refresh_token tokens)})]
      
      ;; Issue our own JWT
      (let [jwt (create-access-token (:id user) (:roles user))]
        (response/redirect (str "/app?token=" jwt))))))
```

---

## ขั้นตอนที่ 2014: Security Headers

```clojure
(ns myapp.security.headers)

;; Comprehensive security headers middleware
(defn wrap-security-headers [handler]
  (fn [request]
    (-> (handler request)
        (assoc-in [:headers "Content-Security-Policy"]
          (str "default-src 'self'; "
               "script-src 'self' 'nonce-{NONCE}'; "
               "style-src 'self' https://fonts.googleapis.com; "
               "font-src 'self' https://fonts.gstatic.com; "
               "img-src 'self' data: https:; "
               "connect-src 'self' https://api.myapp.com; "
               "frame-ancestors 'none'"))
        (assoc-in [:headers "X-Frame-Options"]           "DENY")
        (assoc-in [:headers "X-Content-Type-Options"]    "nosniff")
        (assoc-in [:headers "X-XSS-Protection"]          "1; mode=block")
        (assoc-in [:headers "Referrer-Policy"]           "strict-origin-when-cross-origin")
        (assoc-in [:headers "Permissions-Policy"]
          "camera=(), microphone=(), geolocation=()")
        (assoc-in [:headers "Strict-Transport-Security"]
          "max-age=31536000; includeSubDomains; preload"))))

;; CORS middleware with allowlist
(defn wrap-cors [handler allowed-origins]
  (fn [request]
    (let [origin (get-in request [:headers "origin"])
          allowed? (contains? (set allowed-origins) origin)]
      (if (= :options (:request-method request))
        ;; Preflight
        {:status  204
         :headers (cond-> {"Access-Control-Allow-Methods" "GET,POST,PUT,DELETE,OPTIONS"
                            "Access-Control-Allow-Headers" "Content-Type,Authorization"
                            "Access-Control-Max-Age"       "86400"}
                    allowed? (assoc "Access-Control-Allow-Origin" origin
                                    "Access-Control-Allow-Credentials" "true"))}
        ;; Regular request
        (let [response (handler request)]
          (cond-> response
            allowed? (assoc-in [:headers "Access-Control-Allow-Origin"] origin)
            allowed? (assoc-in [:headers "Access-Control-Allow-Credentials"] "true")))))))
```

---

## ขั้นตอนที่ 2015: Input Validation และ SQL Injection Prevention

```clojure
(ns myapp.security.validation
  (:require [malli.core :as m]
            [malli.error :as me]))

;; Schema validation with Malli
(def UserSchema
  [:map
   [:email    [:and :string [:re #"^[^\s@]+@[^\s@]+\.[^\s@]+$"]]]
   [:password [:and :string [:min-count 8]
                [:re #"(?=.*[A-Z])(?=.*[0-9])"]]]
   [:name     [:and :string [:min-count 2] [:max-count 100]]]
   [:age      {:optional true} [:and :int [:>= 18] [:<= 120]]]])

(defn validate-input [schema data]
  (if (m/validate schema data)
    {:valid? true :data data}
    {:valid?  false
     :errors  (me/humanize (m/explain schema data))}))

;; Prevent XSS: sanitize HTML
(defn sanitize-html [s]
  (-> s
      (clojure.string/replace #"<script[^>]*>.*?</script>" "")
      (clojure.string/replace #"<[^>]+>" "")
      (clojure.string/replace #"&" "&amp;")
      (clojure.string/replace #"<" "&lt;")
      (clojure.string/replace #">" "&gt;")
      (clojure.string/replace #"\"" "&quot;")))

;; SQL injection prevention: ALWAYS use parameterized queries
(defn safe-search-query [db user-input]
  ;; Good: parameterized query
  (jdbc/execute! db
    ["SELECT * FROM products WHERE name ILIKE ?" (str "%" user-input "%")]))

;; Path traversal prevention
(defn safe-file-path [base-dir filename]
  (let [canonical-base (.getCanonicalPath (java.io.File. base-dir))
        canonical-file (.getCanonicalPath (java.io.File. base-dir filename))]
    (when (.startsWith canonical-file canonical-base)
      canonical-file)))

;; Rate limiting by IP
(defonce request-counts (atom {}))

(defn check-ip-rate-limit! [ip max-requests window-seconds]
  (let [now    (System/currentTimeMillis)
        window (* window-seconds 1000)]
    (swap! request-counts
           (fn [counts]
             (let [requests (filter #(> % (- now window))
                                     (get counts ip []))]
               (assoc counts ip (conj requests now)))))
    (< (count (get @request-counts ip [])) max-requests)))
```

---

## Project: Secure API with Full Auth Stack

```clojure
(ns myapp.api.secure
  (:require [reitit.ring :as ring]
            [reitit.ring.middleware.muuntaja :as muuntaja]))

(defn build-secure-app [db redis-pool]
  (ring/ring-handler
    (ring/router
      [["/auth"
        ["/login"   {:post login-handler}]
        ["/refresh" {:post refresh-token-handler}]
        ["/logout"  {:post logout-handler}]]
       
       ["/api"
        {:middleware [wrap-jwt-auth]}
        
        ["/profile"
         {:get (fn [req]
                 {:status 200
                  :body   (get-user-profile db (get-in req [:user :sub]))})}]
        
        ["/admin"
         {:middleware [(require-permission :admin/access)]}
         ["/users" {:get list-users-handler}]]]]
      
      {:data {:middleware [wrap-security-headers
                            wrap-cors
                            muuntaja/format-middleware]}})
    
    (ring/create-default-handler
      {:not-found         (constantly {:status 404 :body {:error "Not found"}})
       :method-not-allowed (constantly {:status 405 :body {:error "Method not allowed"}})})))
```

---

*Part 68 จาก 100+ | ขั้นตอน 2011-2040 จาก 1000+*
