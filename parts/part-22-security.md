# Part 22: Security ใน Clojure
## ขั้นตอนที่ 631-660: Authentication, Authorization, Encryption, OWASP

---

## บทนำ

Security เป็นเรื่องสำคัญในทุก application:
- **Authentication**: ใครเป็นผู้ใช้นี้?
- **Authorization**: ผู้ใช้นี้ทำอะไรได้บ้าง?
- **Encryption**: ปกป้องข้อมูล
- **Input Validation**: ป้องกัน injection
- **OWASP Top 10**: ช่องโหว่ที่พบบ่อย

---

## ขั้นตอนที่ 631: Password Hashing

```clojure
;; deps.edn
;; {:deps {buddy/buddy-hashers {:mvn/version "2.0.167"}}}

(require '[buddy.hashers :as hashers])

;; Hash password ด้วย bcrypt (ปลอดภัยที่สุด)
(def hashed (hashers/derive "my-secret-password"))
;; => "bcrypt+sha512$..."

;; Verify
(hashers/check "my-secret-password" hashed)  ; => true
(hashers/check "wrong-password" hashed)       ; => false

;; argon2 (modern, recommended)
(def argon-hash
  (hashers/derive "password"
    {:alg :argon2id
     :iterations 3
     :memory 65536
     :parallelism 2}))

;; ❌ Never do this:
;; (= (md5 "password") stored-hash)  ; MD5 is NOT secure!
;; (= (sha1 "password") stored-hash) ; SHA1 is NOT secure!

;; ✅ Always use bcrypt/argon2:
(hashers/derive "password" {:alg :bcrypt+sha512 :iterations 12})
```

---

## ขั้นตอนที่ 632: JWT Authentication

```clojure
;; deps.edn
;; {:deps {buddy/buddy-sign {:mvn/version "3.5.351"}}}

(require '[buddy.sign.jwt :as jwt]
         '[buddy.core.keys :as keys])

;; ===== Symmetric JWT (HMAC) =====
(def secret (or (System/getenv "JWT_SECRET")
                (throw (ex-info "JWT_SECRET not set" {}))))

(defn create-access-token [user-id roles]
  (jwt/sign
    {:sub  (str user-id)
     :roles roles
     :iat  (System/currentTimeMillis)
     :exp  (+ (System/currentTimeMillis) (* 15 60 1000))}  ; 15 min
    secret))

(defn create-refresh-token [user-id]
  (jwt/sign
    {:sub  (str user-id)
     :type :refresh
     :iat  (System/currentTimeMillis)
     :exp  (+ (System/currentTimeMillis) (* 7 24 60 60 1000))}  ; 7 days
    secret))

(defn verify-token [token]
  (try
    (let [claims (jwt/unsign token secret)]
      ;; Check expiry
      (when (< (System/currentTimeMillis) (:exp claims))
        claims))
    (catch Exception _
      nil)))

;; ===== Asymmetric JWT (RS256) ===== (preferred for microservices)
(def private-key (keys/private-key "private.pem"))
(def public-key  (keys/public-key  "public.pem"))

(defn create-token-rs256 [claims]
  (jwt/sign claims private-key {:alg :rs256}))

(defn verify-token-rs256 [token]
  (jwt/unsign token public-key {:alg :rs256}))
```

---

## ขั้นตอนที่ 633: OAuth2 Integration

```clojure
(ns myapp.auth.oauth
  (:require [clj-http.client :as http]
            [ring.util.response :as resp]))

;; OAuth2 flow สำหรับ Google login
(def google-config
  {:client-id      (System/getenv "GOOGLE_CLIENT_ID")
   :client-secret  (System/getenv "GOOGLE_CLIENT_SECRET")
   :redirect-uri   "http://localhost:3000/auth/google/callback"
   :auth-url       "https://accounts.google.com/o/oauth2/v2/auth"
   :token-url      "https://oauth2.googleapis.com/token"
   :userinfo-url   "https://www.googleapis.com/oauth2/v3/userinfo"})

;; Step 1: Redirect to Google
(defn google-login-url []
  (str (:auth-url google-config)
       "?client_id=" (:client-id google-config)
       "&redirect_uri=" (:redirect-uri google-config)
       "&response_type=code"
       "&scope=email+profile"
       "&state=" (generate-state-token)))

;; Step 2: Handle callback
(defn handle-google-callback [{:keys [params]}]
  (let [code  (:code params)
        state (:state params)]
    
    ;; Verify state to prevent CSRF
    (when-not (valid-state? state)
      (throw (ex-info "Invalid state" {:status 400})))
    
    ;; Exchange code for tokens
    (let [token-response (http/post (:token-url google-config)
                           {:form-params {:code          code
                                          :client_id     (:client-id google-config)
                                          :client_secret (:client-secret google-config)
                                          :redirect_uri  (:redirect-uri google-config)
                                          :grant_type    "authorization_code"}
                            :as :json})
          access-token (get-in token-response [:body :access_token])
          
          ;; Get user info
          user-info (http/get (:userinfo-url google-config)
                      {:headers {"Authorization" (str "Bearer " access-token)}
                       :as :json})
          google-user (:body user-info)]
      
      ;; Find or create user
      (let [user (or (db/find-user-by-email (:email google-user))
                     (db/create-user! {:name  (:name google-user)
                                        :email (:email google-user)
                                        :provider :google
                                        :provider-id (:sub google-user)}))]
        
        ;; Create session
        {:status 302
         :headers {"Location" "/dashboard"}
         :session {:user-id (:id user)}}))))
```

---

## ขั้นตอนที่ 634: Role-Based Access Control (RBAC)

```clojure
(ns myapp.auth.rbac)

;; Permission definitions
(def permissions
  {:user/read     #{:user :admin :moderator}
   :user/write    #{:admin}
   :user/delete   #{:admin}
   :post/read     #{:user :admin :moderator :guest}
   :post/write    #{:user :admin :moderator}
   :post/delete   #{:admin :moderator}
   :admin/panel   #{:admin}})

(defn has-permission? [user permission]
  (let [user-role (keyword (:role user))
        allowed-roles (get permissions permission #{})]
    (contains? allowed-roles user-role)))

;; Middleware
(defn require-permission [permission handler]
  (fn [request]
    (let [user (:current-user request)]
      (if (has-permission? user permission)
        (handler request)
        {:status 403
         :body {:error "Forbidden"
                 :required-permission permission}}))))

;; Policy-based authorization (more flexible)
(defmulti authorized? (fn [action user resource] action))

(defmethod authorized? :post/delete [_ user post]
  (or (= :admin (:role user))
      (= (:id user) (:author-id post))))

(defmethod authorized? :account/view [_ user account]
  (or (= :admin (:role user))
      (= (:id user) (:user-id account))))

;; Use in handler
(defn delete-post [request]
  (let [user (:current-user request)
        post (db/get-post (-> request :path-params :id))]
    (when-not (authorized? :post/delete user post)
      (throw (ex-info "Forbidden" {:status 403})))
    (db/delete-post! (:id post))
    {:status 204}))
```

---

## ขั้นตอนที่ 635: SQL Injection Prevention

```clojure
;; ❌ DANGEROUS: String concatenation
(defn find-user-bad [email]
  (jdbc/execute! ds
    [(str "SELECT * FROM users WHERE email = '" email "'")]))
;; email = "'; DROP TABLE users; --" → SQL injection!

;; ✅ SAFE: Parameterized queries (always!)
(defn find-user-safe [email]
  (jdbc/execute! ds
    ["SELECT * FROM users WHERE email = ?" email]))

;; ✅ SAFE: HoneySQL
(defn find-user-honey [email]
  (jdbc/execute! ds
    (hsql/format
      {:select [:*] :from :users :where [:= :email email]})))

;; ✅ SAFE: next.jdbc sql/find-by-keys
(defn find-user-jdbc [email]
  (sql/find-by-keys ds :users {:email email}))

;; Rule: NEVER interpolate user input into SQL!
;; Always use ? placeholders or query builders
```

---

## ขั้นตอนที่ 636: XSS Prevention

```clojure
;; Cross-Site Scripting (XSS) prevention

;; deps.edn
;; {:deps {org.owasp.encoder/encoder {:mvn/version "1.2.3"}}}

(import '[org.owasp.encoder Encode])

;; Encode output ก่อนส่งไป HTML
(defn html-encode [s]
  (Encode/forHtml s))

;; ❌ DANGEROUS: raw user input in HTML
(defn dangerous-response [user-name]
  {:status 200
   :body (str "<html><body>Hello " user-name "</body></html>")})
;; user-name = "<script>alert('XSS')</script>" → XSS!

;; ✅ SAFE: encode before rendering
(defn safe-response [user-name]
  {:status 200
   :body (str "<html><body>Hello " (html-encode user-name) "</body></html>")})

;; CSP Header
(defn wrap-security-headers [handler]
  (fn [request]
    (let [response (handler request)]
      (update response :headers merge
              {"Content-Security-Policy"   "default-src 'self'; script-src 'self'"
               "X-Content-Type-Options"   "nosniff"
               "X-Frame-Options"          "DENY"
               "X-XSS-Protection"         "1; mode=block"
               "Strict-Transport-Security" "max-age=31536000; includeSubDomains"
               "Referrer-Policy"          "strict-origin-when-cross-origin"}))))
```

---

## ขั้นตอนที่ 637: Encryption

```clojure
;; deps.edn
;; {:deps {buddy/buddy-core {:mvn/version "1.11.432"}}}

(require '[buddy.core.crypto :as crypto]
         '[buddy.core.codecs :as codecs]
         '[buddy.core.nonce :as nonce])

;; AES-256-GCM Encryption
(defn encrypt [plaintext secret-key]
  (let [key   (codecs/hex->bytes secret-key)  ; 32 bytes = 256 bits
        iv    (nonce/random-bytes 12)           ; GCM nonce
        data  (codecs/str->bytes plaintext)
        cipher (crypto/encrypt data key iv {:algorithm :aes256-gcm})]
    {:ciphertext (codecs/bytes->hex cipher)
     :iv         (codecs/bytes->hex iv)}))

(defn decrypt [{:keys [ciphertext iv]} secret-key]
  (let [key    (codecs/hex->bytes secret-key)
        iv-b   (codecs/hex->bytes iv)
        cipher (codecs/hex->bytes ciphertext)
        data   (crypto/decrypt cipher key iv-b {:algorithm :aes256-gcm})]
    (codecs/bytes->str data)))

;; Encrypt sensitive data before storing in DB
(defn store-sensitive! [db user-id sensitive-data]
  (let [encryption-key (System/getenv "ENCRYPTION_KEY")
        encrypted (encrypt sensitive-data encryption-key)]
    (db/store-encrypted! db user-id encrypted)))

;; Hashing for data integrity
(require '[buddy.core.hash :as hash])
(require '[buddy.core.mac :as mac])

;; HMAC for webhook verification
(defn verify-webhook-signature [payload signature secret]
  (let [expected (mac/hash payload {:key secret :alg :hmac+sha256})]
    (= signature (codecs/bytes->hex expected))))
```

---

## ขั้นตอนที่ 638: Rate Limiting (Security)

```clojure
(ns myapp.security.rate-limit
  (:require [clojure.core.cache.wrapped :as cw]))

;; IP-based rate limiting สำหรับ login endpoint
(def login-attempts
  (cw/ttl-cache-factory {} :ttl 900000))  ; 15 min window

(defn check-login-rate-limit! [ip]
  (let [attempts (cw/lookup login-attempts ip)]
    (when (and attempts (>= attempts 10))
      (throw (ex-info "Too many login attempts"
                       {:status 429
                        :retry-after 900})))))

(defn record-login-attempt! [ip success?]
  (if success?
    ;; Reset on success
    (swap! login-attempts dissoc ip)
    ;; Increment on failure
    (swap! login-attempts update ip (fnil inc 0))))

;; CAPTCHA after N failures
(defn login-handler [request]
  (let [ip    (:remote-addr request)
        creds (:body-params request)]
    
    (check-login-rate-limit! ip)
    
    (if-let [user (auth/authenticate (:email creds) (:password creds))]
      (do
        (record-login-attempt! ip true)
        {:status 200 :body {:token (create-token (:id user) (:roles user))}})
      (do
        (record-login-attempt! ip false)
        {:status 401 :body {:error "Invalid credentials"}}))))
```

---

## สรุป Security Best Practices

```
OWASP Top 10 Protection:
========================
1. Broken Access Control
   ✓ Check permissions per request
   ✓ RBAC or ABAC
   ✓ Deny by default

2. Cryptographic Failures  
   ✓ bcrypt/argon2 for passwords
   ✓ TLS/HTTPS only
   ✓ AES-256 for sensitive data
   ✗ Never MD5/SHA1 for passwords

3. Injection (SQL/NoSQL/Command)
   ✓ Parameterized queries ALWAYS
   ✓ Input validation with Malli/Spec
   ✗ Never string concat with user input

4. Insecure Design
   ✓ Threat modeling
   ✓ Principle of least privilege

5. Security Misconfiguration
   ✓ Security headers
   ✓ Error messages (don't leak info)
   ✓ Config from environment vars

6. Vulnerable Components
   ✓ Regular dependency updates
   ✓ CVE monitoring (Snyk, Dependabot)

7. Authentication Failures
   ✓ Rate limiting on login
   ✓ Multi-factor authentication
   ✓ Secure token storage

8. Integrity Failures
   ✓ HMAC for webhooks
   ✓ Code signing

9. Logging Failures  
   ✓ Log auth events
   ✓ Don't log passwords/tokens

10. SSRF
    ✓ Validate URLs
    ✓ Network egress controls
```

---

*Part 22 จาก 100+ | ขั้นตอน 631-660 จาก 1000+*
