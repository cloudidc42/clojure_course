# Part 70: Multi-tenancy Patterns
## ขั้นตอนที่ 2071-2100: Schema Isolation, Row-Level Security, Tenant Resolution

---

## บทนำ

Multi-tenancy architectures:
- **Schema-per-tenant** - PostgreSQL schema isolation
- **Row-level security** - data separation in shared tables
- **Tenant resolution** - subdomain, header, JWT claim
- **Tenant-aware middleware** - inject tenant context
- **Provisioning** - automated tenant onboarding

---

## ขั้นตอนที่ 2071: Tenant Context

```clojure
(ns myapp.multitenancy.context)

;; Thread-local tenant context
(def ^:dynamic *tenant* nil)

(defmacro with-tenant [tenant-id & body]
  `(binding [*tenant* ~tenant-id]
     ~@body))

(defn current-tenant []
  (or *tenant*
      (throw (ex-info "No tenant in context" {}))))

;; Tenant resolution strategies
(defmulti resolve-tenant :strategy)

;; Strategy 1: Subdomain (acme.myapp.com -> acme)
(defmethod resolve-tenant :subdomain [{:keys [request]}]
  (let [host   (get-in request [:headers "host"] "")
        parts  (clojure.string/split host #"\.")
        domain (take-last 2 parts)  ; myapp.com
        sub    (drop-last 2 parts)]
    (when (seq sub) (first sub))))

;; Strategy 2: Custom header
(defmethod resolve-tenant :header [{:keys [request header-name]}]
  (get-in request [:headers (or header-name "x-tenant-id")]))

;; Strategy 3: JWT claim
(defmethod resolve-tenant :jwt-claim [{:keys [request claim-name]}]
  (get-in request [:user (or claim-name :tenant-id)]))

;; Strategy 4: Path prefix (/tenant/{id}/api/...)
(defmethod resolve-tenant :path [{:keys [request]}]
  (second (re-matches #"/tenant/([^/]+).*" (:uri request))))

;; Tenant resolution middleware
(defn wrap-tenant-context [handler resolver-config]
  (fn [request]
    (let [tenant-id (resolve-tenant (assoc resolver-config :request request))]
      (if tenant-id
        (with-tenant tenant-id
          (handler (assoc request :tenant-id tenant-id)))
        {:status 400
         :body   {:error "Cannot determine tenant"}}))))
```

---

## ขั้นตอนที่ 2072: Schema-Per-Tenant Isolation

```clojure
(ns myapp.multitenancy.schema
  (:require [next.jdbc :as jdbc]))

;; Set search_path for PostgreSQL schema isolation
(defn with-tenant-schema [db tenant-id f]
  (jdbc/with-transaction [tx db]
    (jdbc/execute! tx
      [(str "SET search_path TO tenant_" tenant-id ", public")])
    (f tx)))

;; Macro convenience
(defmacro in-tenant-schema [db tenant-id & body]
  `(with-tenant-schema ~db ~tenant-id
     (fn [tx#]
       (binding [myapp.db/*db* tx#]
         ~@body))))

;; Provision new tenant schema
(defn provision-tenant-schema! [admin-db tenant-id]
  (let [schema-name (str "tenant_" tenant-id)]
    (jdbc/execute! admin-db
      [(str "CREATE SCHEMA IF NOT EXISTS " schema-name)])
    
    ;; Copy table structure from template
    (doseq [table ["users" "products" "orders" "settings"]]
      (jdbc/execute! admin-db
        [(str "CREATE TABLE " schema-name "." table
              " (LIKE public." table " INCLUDING ALL)")]))
    
    ;; Create tenant-specific indexes
    (jdbc/execute! admin-db
      [(str "CREATE INDEX " schema-name "_users_email_idx"
            " ON " schema-name ".users(email)")])
    
    (println "Provisioned schema for tenant:" tenant-id)
    schema-name))

;; Tenant middleware with schema setting
(defn wrap-tenant-schema [handler db]
  (fn [request]
    (if-let [tenant-id (:tenant-id request)]
      (in-tenant-schema db tenant-id
        (handler request))
      (handler request))))
```

---

## ขั้นตอนที่ 2073: Row-Level Security

```clojure
;; Row-Level Security (RLS) in PostgreSQL
;; Shared tables with tenant_id column

;; SQL: Enable RLS
;; ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
;; CREATE POLICY tenant_isolation ON orders
;;   USING (tenant_id = current_setting('app.tenant_id')::uuid);

;; Set tenant context in connection
(defn set-tenant-context! [db tenant-id]
  (jdbc/execute! db
    [(str "SET LOCAL app.tenant_id = '" tenant-id "'")]))

;; Queries automatically filtered by RLS!
(defn get-orders [db]
  ;; No WHERE tenant_id = ? needed - RLS handles it
  (jdbc/execute! db ["SELECT * FROM orders"]))

;; Tenant-aware connection
(defn with-rls-tenant [db tenant-id f]
  (jdbc/with-transaction [tx db]
    (set-tenant-context! tx tenant-id)
    (f tx)))

;; Insert automatically includes tenant_id via trigger
(defn create-order! [db order-data]
  ;; No need to specify tenant_id - set via current_setting
  (jdbc/execute-one! db
    ["INSERT INTO orders (total, items) VALUES (?, ?::jsonb) RETURNING *"
     (:total order-data)
     (json/generate-string (:items order-data))]))

;; Bypass RLS for admin operations (use carefully!)
(defn admin-query [admin-db sql params]
  (jdbc/execute! admin-db
    (into [sql] params)
    ;; admin role bypasses RLS
    ))
```

---

## ขั้นตอนที่ 2074: Tenant Configuration

```clojure
(ns myapp.multitenancy.config)

;; Tenant model
(defrecord Tenant [id name plan features limits])

;; Plans with features
(def plans
  {:starter {:features #{:basic-reports :api-access}
              :limits   {:users 5 :api-calls-per-month 10000 :storage-gb 1}}
   
   :pro     {:features #{:basic-reports :api-access :advanced-analytics
                          :webhooks :custom-domain}
              :limits   {:users 50 :api-calls-per-month 100000 :storage-gb 10}}
   
   :enterprise {:features #{:all}
                :limits   {:users :unlimited :api-calls-per-month :unlimited
                            :storage-gb 1000}}})

;; Feature flags per tenant
(defn tenant-has-feature? [tenant-id feature]
  (if-let [tenant (get-tenant tenant-id)]
    (let [plan-features (get-in plans [(:plan tenant) :features] #{})]
      (or (contains? plan-features feature)
          (contains? plan-features :all)))
    false))

;; Tenant-scoped rate limiting
(defn check-tenant-rate-limit! [redis-pool tenant-id resource]
  (let [tenant  (get-tenant tenant-id)
        limit   (get-in plans [(:plan tenant) :limits resource] 0)
        key     (str "tenant:" tenant-id ":usage:" (name resource)
                      ":" (java.time.YearMonth/now))]
    (when (not= :unlimited limit)
      (let [current (car/wcar redis-pool
                       (car/incr key))]
        (when (= current 1)
          (car/wcar redis-pool
            (car/expireat key (seconds-until-end-of-month))))
        (when (> current limit)
          (throw (ex-info "Rate limit exceeded"
                           {:tenant tenant-id
                            :resource resource
                            :limit limit
                            :current current})))
        current))))
```

---

## ขั้นตอนที่ 2075: Tenant Provisioning

```clojure
(ns myapp.multitenancy.provisioning)

;; Complete tenant onboarding workflow
(defn provision-tenant! [admin-db redis-pool tenant-data]
  (let [tenant-id (str (java.util.UUID/randomUUID))
        slug      (-> (:name tenant-data)
                      clojure.string/lower-case
                      (clojure.string/replace #"[^a-z0-9]" "-"))]
    
    ;; 1. Create tenant record
    (jdbc/execute-one! admin-db
      ["INSERT INTO tenants (id, slug, name, plan, created_at)
        VALUES (?, ?, ?, ?, NOW())
        RETURNING *"
       tenant-id slug (:name tenant-data)
       (name (get tenant-data :plan :starter))])
    
    ;; 2. Provision database schema
    (provision-tenant-schema! admin-db tenant-id)
    
    ;; 3. Create admin user for tenant
    (in-tenant-schema admin-db tenant-id
      (fn [tx]
        (create-user! tx
          {:email    (:admin-email tenant-data)
           :name     (:admin-name tenant-data)
           :roles    [:admin]
           :tenant-id tenant-id})))
    
    ;; 4. Initialize tenant settings
    (car/wcar redis-pool
      (car/set (str "tenant:" tenant-id ":config")
                (json/generate-string
                  {:plan     (:plan tenant-data :starter)
                   :features (get-in plans [(:plan tenant-data :starter) :features])})))
    
    ;; 5. Send welcome email
    (send-email!
      {:to      (:admin-email tenant-data)
       :subject "Welcome to MyApp!"
       :body    (render-welcome-email tenant-id (:name tenant-data))})
    
    (println "Tenant provisioned:" tenant-id)
    {:tenant-id tenant-id
     :slug      slug
     :login-url (str "https://" slug ".myapp.com/login")}))

;; Deprovision (GDPR compliance)
(defn deprovision-tenant! [admin-db tenant-id]
  ;; 1. Archive data
  (archive-tenant-data! admin-db tenant-id)
  
  ;; 2. Drop schema
  (jdbc/execute! admin-db
    [(str "DROP SCHEMA tenant_" tenant-id " CASCADE")])
  
  ;; 3. Mark tenant as deleted
  (jdbc/execute-one! admin-db
    ["UPDATE tenants SET deleted_at = NOW() WHERE id = ?" tenant-id])
  
  (println "Tenant deprovisioned:" tenant-id))
```

---

## Project: Multi-tenant SaaS Scaffold

```clojure
(ns myapp.multitenant-app)

(defn build-multitenant-app [admin-db redis-pool]
  (ring/ring-handler
    (ring/router
      ;; Public routes (no tenant)
      [["/tenants"
        {:post (fn [req]
                 {:status 201
                  :body   (provision-tenant!
                            admin-db redis-pool
                            (:body-params req))})}]
       
       ;; Tenant routes
       ["/api"
        {:middleware
         [(wrap-tenant-context {:strategy :subdomain})
          (partial wrap-tenant-schema admin-db)
          wrap-jwt-auth
          (fn [handler]
            (fn [req]
              ;; Check feature flag
              (when-not (tenant-has-feature?
                          (:tenant-id req)
                          (route-required-feature req))
                (throw (ex-info "Feature not available on your plan"
                                 {:status 402})))
              (handler req)))]}
        
        ["/orders"
         {:get  (fn [req] {:status 200 :body (get-orders *db*)})
          :post (fn [req] {:status 201 :body (create-order! *db* (:body-params req))})}]]])))
```

---

*Part 70 จาก 100+ | ขั้นตอน 2071-2100 จาก 1000+*
