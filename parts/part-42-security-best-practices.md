# Part 42: Security Best Practices
## ขั้นตอนที่ 1231-1260: Authentication, Authorization, OWASP, Cryptography

---

## บทนำ

Security สำหรับ Clojure applications:
- **Authentication** - JWT, OAuth2, MFA
- **Authorization** - RBAC, ABAC, policies
- **OWASP Top 10** - SQL injection, XSS, CSRF prevention
- **Cryptography** - hashing, encryption, signing
- **Secrets Management** - Vault, environment variables

---

## ขั้นตอนที่ 1231: JWT Authentication

```clojure
;; deps.edn
;; {:deps {buddy/buddy-sign    {:mvn/version "3.5.351"}
;;         buddy/buddy-hashers {:mvn/version "2.0.167"}
;;         buddy/buddy-auth    {:mvn/version "3.0.323"}}}

(ns myapp.auth.jwt
  (:require [buddy.sign.jwt :as jwt]
            [buddy.hashers :as hashers]))

(def secret (System/getenv "JWT_SECRET"))
(def access-token-ttl  (* 15 60))      ; 15 minutes
(def refresh-token-ttl (* 7 24 3600))  ; 7 days

;; Generate tokens
(defn generate-tokens [user]
  (let [now  (System/currentTimeMillis)
        base {:iss "myapp"
               :iat (long (/ now 1000))
               :sub (str (:id user))
               :uid (str (:id user))
               :email (:email user)
               :role  (:role user)}]
    {:access-token  (jwt/sign (assoc base :exp (+ (long (/ now 1000))
                                                    access-token-ttl))
                               secret {:alg :hs256})
     :refresh-token (jwt/sign (assoc base :type "refresh"
                                          :exp  (+ (long (/ now 1000))
                                                    refresh-token-ttl))
                               secret {:alg :hs256})}))

;; Verify token
(defn verify-token [token]
  (try
    {:ok (jwt/unsign token secret {:alg :hs256})}
    (catch clojure.lang.ExceptionInfo e
      {:error (ex-message e)})))

;; Password hashing (bcrypt)
(defn hash-password [password]
  (hashers/derive password {:alg :bcrypt+sha512}))

(defn verify-password [password hash]
  (hashers/check password hash))

;; Auth middleware
(defn wrap-jwt-auth [handler {:keys [public-paths]}]
  (fn [request]
    (if (some #(re-matches (re-pattern %) (:uri request)) public-paths)
      (handler request)
      (let [token (some-> (get-in request [:headers "authorization"])
                           (#(when (clojure.string/starts-with? % "Bearer ")
                               (subs % 7))))]
        (if-not token
          {:status 401 :body {:error "Authentication required"}}
          (let [{:keys [ok error]} (verify-token token)]
            (if error
              {:status 401 :body {:error error}}
              (handler (assoc request :auth ok)))))))))
```

---

## ขั้นตอนที่ 1232: RBAC Authorization

```clojure
(ns myapp.auth.rbac)

;; Role definitions with permissions
(def roles
  {:guest  #{:product/read}
   :user   #{:product/read
              :order/create :order/read-own
              :cart/write :cart/read}
   :seller #{:product/read :product/create :product/update-own
              :order/read-own}
   :admin  #{:product/read :product/create :product/update :product/delete
              :order/read :order/update :order/cancel
              :user/read :user/update :user/delete}})

;; Hierarchy: admin includes all
(def role-hierarchy
  {:admin  [:admin :seller :user :guest]
   :seller [:seller :user :guest]
   :user   [:user :guest]
   :guest  [:guest]})

(defn has-permission? [user permission]
  (let [user-roles (mapcat (fn [r] (get role-hierarchy r [r]))
                             (:roles user))
        permissions (into #{} (mapcat #(get roles % #{}) user-roles))]
    (contains? permissions permission)))

;; Resource-level authorization
(defn can-access-resource? [user resource action]
  (case action
    :read   (or (has-permission? user (keyword (name (:type resource)) "read"))
                (has-permission? user (keyword (name (:type resource)) "read-own")))
    :update (or (has-permission? user (keyword (name (:type resource)) "update"))
                (and (has-permission? user (keyword (name (:type resource)) "update-own"))
                     (= (:owner-id resource) (:id user))))
    :delete (has-permission? user (keyword (name (:type resource)) "delete"))
    false))

;; Middleware
(defn wrap-authorize [handler permission]
  (fn [request]
    (let [user (:auth request)]
      (if (has-permission? user permission)
        (handler request)
        {:status 403 :body {:error "Insufficient permissions"}}))))

;; Usage
(defroutes protected-routes
  (GET "/admin/users" req
    (-> req
        ((wrap-authorize identity :user/read))
        list-users-handler)))
```

---

## ขั้นตอนที่ 1233: SQL Injection Prevention

```clojure
(ns myapp.db.safe
  (:require [honey.sql :as sql]
            [next.jdbc :as jdbc]))

;; ===== DANGEROUS: String interpolation =====
;; NEVER do this!
(defn bad-find-user [username]
  (jdbc/execute! ds
    [(str "SELECT * FROM users WHERE username = '" username "'")]))
;; Attack: username = "'; DROP TABLE users; --"

;; ===== SAFE: Parameterized queries =====
(defn find-user [ds username]
  (jdbc/execute-one! ds
    ["SELECT * FROM users WHERE username = ?" username]))

;; HoneySQL for dynamic queries (always safe)
(defn find-users-filtered [ds {:keys [status role min-age]}]
  (let [query (cond-> {:select [:*]
                         :from   [:users]}
                 status  (update :where (fnil conj [:and]) [:= :status status])
                 role    (update :where (fnil conj [:and]) [:= :role role])
                 min-age (update :where (fnil conj [:and]) [:>= :age min-age]))]
    (jdbc/execute! ds (sql/format query))))

;; ===== Input validation before any DB operation =====
(defn validate-sort-column [col allowed-columns]
  (when-not (contains? allowed-columns col)
    (throw (ex-info "Invalid sort column"
                     {:column col :allowed allowed-columns})))
  col)

;; Safe search with full-text
(defn search-products [ds query]
  (jdbc/execute! ds
    ["SELECT * FROM products
      WHERE to_tsvector('english', name || ' ' || description)
      @@ plainto_tsquery('english', ?)
      ORDER BY ts_rank(to_tsvector('english', name), plainto_tsquery('english', ?)) DESC"
     query query]))
```

---

## ขั้นตอนที่ 1234: XSS Prevention

```clojure
(ns myapp.security.xss
  (:import [org.owasp.encoder Encode]))

;; Encode user-provided content before rendering
(defn encode-html [s]
  (Encode/forHtml s))

(defn encode-html-attr [s]
  (Encode/forHtmlAttribute s))

(defn encode-js [s]
  (Encode/forJavaScript s))

(defn encode-url [s]
  (Encode/forUriComponent s))

;; Content Security Policy headers
(defn csp-header []
  (clojure.string/join "; "
    ["default-src 'self'"
     "script-src 'self' cdn.jsdelivr.net"
     "style-src 'self' 'unsafe-inline' fonts.googleapis.com"
     "img-src 'self' data: https:"
     "font-src 'self' fonts.gstatic.com"
     "connect-src 'self'"
     "frame-ancestors 'none'"
     "upgrade-insecure-requests"]))

;; Security headers middleware
(defn wrap-security-headers [handler]
  (fn [request]
    (let [response (handler request)]
      (update response :headers merge
        {"Content-Security-Policy"    (csp-header)
         "X-Content-Type-Options"     "nosniff"
         "X-Frame-Options"            "DENY"
         "X-XSS-Protection"           "1; mode=block"
         "Referrer-Policy"            "strict-origin-when-cross-origin"
         "Permissions-Policy"         "camera=(), microphone=(), geolocation=()"
         "Strict-Transport-Security"  "max-age=31536000; includeSubDomains"}))))
```

---

## ขั้นตอนที่ 1235: CSRF Protection

```clojure
(ns myapp.security.csrf
  (:require [ring.middleware.anti-forgery :as csrf]))

;; Ring anti-forgery middleware
(defn wrap-csrf [handler]
  (csrf/wrap-anti-forgery handler
    {:error-handler
     (fn [request]
       {:status 403
        :headers {"Content-Type" "application/json"}
        :body (cheshire.core/generate-string
                {:error "CSRF token invalid"})})}))

;; For SPA/API: use double-submit cookie pattern
(defn generate-csrf-token []
  (let [bytes (byte-array 32)]
    (.nextBytes (java.security.SecureRandom.) bytes)
    (java.util.Base64/getEncoder)
    (.encodeToString (java.util.Base64/getEncoder) bytes)))

(defn wrap-csrf-cookie [handler]
  (fn [request]
    (let [existing-token (get-in request [:cookies "XSRF-TOKEN" :value])
          token          (or existing-token (generate-csrf-token))]
      (if (and (not= (:request-method request) :get)
               (not= token (get-in request [:headers "x-xsrf-token"])))
        {:status 403 :body {:error "CSRF validation failed"}}
        (-> (handler request)
            (assoc-in [:cookies "XSRF-TOKEN"]
                       {:value token :http-only false :same-site :strict}))))))
```

---

## ขั้นตอนที่ 1236: Secrets Management ด้วย HashiCorp Vault

```clojure
;; deps.edn
;; {:deps {vault-clj/vault-clj {:mvn/version "1.1.2"}}}

(ns myapp.secrets
  (:require [vault.client.http :as vault-http]
            [vault.auth.token :as vault-token]))

(defn create-vault-client []
  (vault-http/http-client
    {:address   (or (System/getenv "VAULT_ADDR") "http://vault:8200")
     :auth-type :token
     :token     (System/getenv "VAULT_TOKEN")}))

(def vault (create-vault-client))

;; Read secrets
(defn get-secret [path]
  (vault/read-secret vault path))

(defn get-db-password []
  (get-in (get-secret "secret/data/database") [:data :password]))

(defn get-jwt-secret []
  (get-in (get-secret "secret/data/app") [:data :jwt-secret]))

;; Dynamic secrets (database credentials)
(defn get-db-lease []
  (vault/read-secret vault "database/creds/my-role"))

;; Rotate secrets periodically
(defn start-secret-rotation! []
  (future
    (loop []
      (Thread/sleep (* 60 60 1000))  ; rotate hourly
      (let [new-creds (get-db-lease)]
        (db/rotate-connection! (:username new-creds) (:password new-creds)))
      (recur))))

;; For development: use env vars as fallback
(defn get-config-value [vault-path env-var]
  (or (try
        (get-secret vault-path)
        (catch Exception _ nil))
      (System/getenv env-var)
      (throw (ex-info (str "Config not found: " env-var)
                       {:path vault-path :env-var env-var}))))
```

---

## ขั้นตอนที่ 1237: Input Validation ด้วย Malli

```clojure
(ns myapp.validation
  (:require [malli.core :as m]
            [malli.error :as me]
            [malli.transform :as mt]))

;; Strict schemas with custom validators
(def UserRegistrationSchema
  [:map
   [:email    [:and :string
               [:re {:error/message "Invalid email format"}
                #".+@.+\..+"]]]
   [:password [:and :string
               [:min-count {:error/message "Password must be at least 8 characters"} 8]
               [:re {:error/message "Must contain uppercase letter"}
                #".*[A-Z].*"]
               [:re {:error/message "Must contain number"}
                #".*[0-9].*"]
               [:re {:error/message "Must contain special character"}
                #".*[!@#$%^&*].*"]]]
   [:name     [:string {:min 1 :max 100}]]
   [:phone    {:optional true}
    [:re #"\+?[0-9]{10,15}"]]])

;; Validation middleware
(defn validate-body [schema handler]
  (fn [request]
    (let [data (:body-params request)]
      (if (m/validate schema data)
        (handler (assoc request :validated-body
                          (m/coerce schema data mt/json-transformer)))
        {:status 422
         :body   {:error   "Validation failed"
                   :details (-> (m/explain schema data)
                                 me/humanize)}}))))

;; Safe coercion
(def CoerceSchema
  (m/schema
    [:map
     [:id    :uuid]
     [:price [:and :double [:> 0]]]
     [:tags  [:vector :string]]]))

(defn coerce-input [schema data]
  (try
    {:ok (m/coerce schema data (mt/composite-transformer
                                   mt/json-transformer
                                   mt/strip-extra-keys-transformer))}
    (catch Exception e
      {:error (ex-message e)})))
```

---

## Project: Secure API

```clojure
(ns myapp.secure-api
  (:require [myapp.auth.jwt :as jwt]
            [myapp.auth.rbac :as rbac]
            [myapp.security.xss :as xss]
            [myapp.security.csrf :as csrf]
            [myapp.validation :as v]))

;; Complete security middleware stack
(defn create-secure-handler [routes]
  (-> routes
      ;; Input validation (innermost)
      
      ;; Authorization
      (rbac/wrap-authorize-routes)
      
      ;; Authentication
      (jwt/wrap-jwt-auth {:public-paths ["/api/auth/.*" "/health"]})
      
      ;; CSRF protection
      (csrf/wrap-csrf-cookie)
      
      ;; Security headers
      (xss/wrap-security-headers)
      
      ;; Rate limiting
      (rate-limit/wrap-rate-limit redis-conn
        {:limit  100
         :window 60
         :key-fn #(or (get-in % [:auth :uid])
                       (get-in % [:headers "x-forwarded-for"])
                       (:remote-addr %))})
      
      ;; CORS
      (ring.middleware.cors/wrap-cors
        :access-control-allow-origin  [#"https://myapp.com"]
        :access-control-allow-methods [:get :post :put :delete]
        :access-control-allow-headers ["Content-Type" "Authorization" "X-XSRF-Token"])))
```

---

*Part 42 จาก 100+ | ขั้นตอน 1231-1260 จาก 1000+*
