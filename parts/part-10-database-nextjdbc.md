# Part 10: Database ด้วย next.jdbc และ HoneySQL
## ขั้นตอนที่ 271-300: SQL, ORM, Migrations, Connection Pooling

---

## บทนำ

Clojure ใช้ SQL ตรงๆ แทน ORM เพราะ:
- SQL เป็น language ที่ชัดเจนอยู่แล้ว
- ไม่มี "impedance mismatch" ที่ซับซ้อน
- ง่ายต่อการ optimize
- **next.jdbc** = modern, performant JDBC wrapper
- **HoneySQL** = SQL as Clojure data structures

---

## ขั้นตอนที่ 271: Setup Database

```clojure
;; deps.edn
;; {:deps {com.github.seancorfield/next.jdbc {:mvn/version "1.3.909"}
;;         com.github.seancorfield/honeysql {:mvn/version "2.6.1126"}
;;         org.postgresql/postgresql {:mvn/version "42.7.3"}
;;         com.zaxxer/HikariCP {:mvn/version "5.1.0"}}}

(ns myapp.db
  (:require [next.jdbc :as jdbc]
            [next.jdbc.sql :as sql]
            [next.jdbc.result-set :as rs]))

;; Simple datasource
(def db-spec
  {:dbtype   "postgresql"
   :dbname   "myapp_db"
   :host     "localhost"
   :port     5432
   :user     "dbuser"
   :password "password"})

(def ds (jdbc/get-datasource db-spec))

;; Test connection
(jdbc/execute-one! ds ["SELECT 1 as alive"])
;; => {:alive 1}
```

---

## ขั้นตอนที่ 272: Connection Pool ด้วย HikariCP

```clojure
(ns myapp.db.pool
  (:require [next.jdbc :as jdbc])
  (:import [com.zaxxer.hikari HikariDataSource HikariConfig]))

(defn create-pool [{:keys [host port dbname user password
                            pool-size min-idle]}]
  (let [config (doto (HikariConfig.)
                 (.setJdbcUrl (str "jdbc:postgresql://" host ":" port "/" dbname))
                 (.setUsername user)
                 (.setPassword password)
                 (.setMaximumPoolSize (or pool-size 10))
                 (.setMinimumIdle (or min-idle 2))
                 (.setConnectionTimeout 30000)   ; 30s
                 (.setIdleTimeout 600000)         ; 10min
                 (.setMaxLifetime 1800000)         ; 30min
                 (.setPoolName "myapp-pool"))]
    (HikariDataSource. config)))

;; Usage
(def pool
  (create-pool {:host "localhost"
                :port 5432
                :dbname "myapp_db"
                :user "dbuser"
                :password "password"
                :pool-size 20
                :min-idle 5}))

;; ใช้ pool แทน ds
(jdbc/execute! pool ["SELECT COUNT(*) FROM users"])

;; Close pool เมื่อ shutdown
(.close pool)
```

---

## ขั้นตอนที่ 273: Basic CRUD Operations

```clojure
(ns myapp.db.users
  (:require [next.jdbc :as jdbc]
            [next.jdbc.sql :as sql]))

;; ===== CREATE =====
(defn create-user! [ds user]
  (sql/insert! ds :users
    {:name (:name user)
     :email (:email user)
     :password_hash (:password-hash user)
     :created_at (java.time.Instant/now)}
    {:return-keys true}))  ; return generated keys

;; ===== READ =====
(defn find-user-by-id [ds id]
  (sql/get-by-id ds :users id))

(defn find-user-by-email [ds email]
  (sql/find-by-keys ds :users {:email email}
    {:row-fn #(dissoc % :users/password_hash)}))  ; hide password

(defn find-all-users [ds]
  (sql/find-by-keys ds :users :all
    {:order-by [:name]}))

;; ===== UPDATE =====
(defn update-user! [ds id updates]
  (sql/update! ds :users updates {:id id}
    {:return-keys true}))

;; ===== DELETE =====
(defn delete-user! [ds id]
  (sql/delete! ds :users {:id id}))

;; ===== UPSERT =====
(defn upsert-user! [ds user]
  (jdbc/execute-one! ds
    ["INSERT INTO users (email, name) VALUES (?, ?)
      ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name
      RETURNING *"
     (:email user) (:name user)]))
```

---

## ขั้นตอนที่ 274: Raw SQL Queries

```clojure
;; execute! - multiple results
(defn find-active-users [ds]
  (jdbc/execute! ds
    ["SELECT u.*, COUNT(o.id) as order_count
      FROM users u
      LEFT JOIN orders o ON o.user_id = u.id
      WHERE u.active = true
      GROUP BY u.id
      ORDER BY order_count DESC"]))

;; execute-one! - single result
(defn get-user-with-stats [ds user-id]
  (jdbc/execute-one! ds
    ["SELECT u.*,
             COUNT(DISTINCT o.id) as total_orders,
             COALESCE(SUM(o.total_amount), 0) as total_spent
      FROM users u
      LEFT JOIN orders o ON o.user_id = u.id
      WHERE u.id = ?
      GROUP BY u.id"
     user-id]))

;; Parameterized queries (ALWAYS use params, never string concat!)
(defn search-users [ds {:keys [name email status page size]}]
  (let [page (or page 1)
        size (or size 20)]
    (jdbc/execute! ds
      [(str "SELECT * FROM users WHERE 1=1"
            (when name  " AND name ILIKE ?")
            (when email " AND email = ?")
            (when status " AND status = ?")
            " ORDER BY created_at DESC"
            " LIMIT ? OFFSET ?")
       ;; Dynamic params
       (cond-> []
         name   (conj (str "%" name "%"))
         email  (conj email)
         status (conj status)
         true   (conj size)
         true   (conj (* (dec page) size)))])))
```

---

## ขั้นตอนที่ 275: HoneySQL - SQL as Data

```clojure
(ns myapp.db.queries
  (:require [honey.sql :as sql]
            [honey.sql.helpers :as h]
            [next.jdbc :as jdbc]))

;; Build queries as data!
(def find-users-query
  {:select [:id :name :email :created-at]
   :from [:users]
   :where [:= :active true]
   :order-by [[:created-at :desc]]
   :limit 20})

;; Convert to SQL string
(sql/format find-users-query)
;; => ["SELECT id, name, email, created_at FROM users WHERE active = ? ORDER BY created_at DESC LIMIT ?" true 20]

;; Execute
(defn find-active-users [ds]
  (jdbc/execute! ds (sql/format find-users-query)))

;; Dynamic query building
(defn build-user-query [{:keys [name status role page size]
                          :or {page 1 size 20}}]
  (cond-> (h/select :u.* [:count :o.id :order-count])
    true   (h/from [:users :u])
    true   (h/left-join [:orders :o] [:= :o.user_id :u.id])
    name   (h/where [:ilike :u.name (str "%" name "%")])
    status (h/where [:= :u.status status])
    role   (h/where [:= :u.role role])
    true   (h/group-by :u.id)
    true   (h/order-by [:order-count :desc])
    true   (h/limit size)
    true   (h/offset (* (dec page) size))))

;; ใช้งาน
(defn search-users [ds filters]
  (jdbc/execute! ds (sql/format (build-user-query filters))))
```

---

## ขั้นตอนที่ 276: Transactions

```clojure
;; Simple transaction
(defn transfer-money! [ds from-account to-account amount]
  (jdbc/with-transaction [tx ds]
    ;; All operations use tx (not ds)
    (let [from (sql/get-by-id tx :accounts from-account)
          to   (sql/get-by-id tx :accounts to-account)]
      
      (when (< (:accounts/balance from) amount)
        (throw (ex-info "Insufficient funds"
                        {:status 422
                         :from-balance (:accounts/balance from)
                         :requested amount})))
      
      ;; Debit
      (sql/update! tx :accounts
                   {:balance (- (:accounts/balance from) amount)
                    :updated_at (java.time.Instant/now)}
                   {:id from-account})
      
      ;; Credit  
      (sql/update! tx :accounts
                   {:balance (+ (:accounts/balance to) amount)
                    :updated_at (java.time.Instant/now)}
                   {:id to-account})
      
      ;; Log transaction
      (sql/insert! tx :transactions
                   {:from_account from-account
                    :to_account to-account
                    :amount amount
                    :created_at (java.time.Instant/now)})
      
      {:success true :amount amount})))

;; Nested transactions (savepoints)
(defn complex-operation! [ds user-id data]
  (jdbc/with-transaction [tx ds]
    (try
      (let [result1 (operation-1! tx user-id)
            result2 (operation-2! tx result1 data)]
        {:success true :result result2})
      (catch Exception e
        ;; Transaction auto-rollbacks on exception!
        (throw e)))))
```

---

## ขั้นตอนที่ 277: Database Migrations ด้วย Flyway

```clojure
;; deps.edn
;; {:deps {com.flyway/flyway-core {:mvn/version "9.22.3"}
;;         com.flyway/flyway-database-postgresql {:mvn/version "9.22.3"}}}

(ns myapp.db.migrations
  (:import [org.flywaydb.core Flyway]))

(defn run-migrations! [db-url user password]
  (let [flyway (-> (Flyway/configure)
                   (.dataSource db-url user password)
                   (.locations (into-array String ["classpath:db/migrations"]))
                   (.baselineOnMigrate true)
                   (.load))]
    (.migrate flyway)))

;; Migration files (resources/db/migrations/)
;; V1__create_users.sql
;; V2__add_orders.sql
;; V3__add_indexes.sql
```

```sql
-- resources/db/migrations/V1__create_users.sql
CREATE TABLE users (
    id         BIGSERIAL PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    email      VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role       VARCHAR(50) DEFAULT 'user',
    active     BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_active ON users(active) WHERE active = true;

-- Trigger for updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

---

## ขั้นตอนที่ 278: Result Set Transformation

```clojure
;; next.jdbc คืนค่า namespaced keywords
(jdbc/execute-one! ds ["SELECT id, name FROM users WHERE id = 1"])
;; => {:users/id 1, :users/name "สมชาย"}

;; Unqualified keywords
(jdbc/execute-one! ds ["SELECT id, name FROM users WHERE id = 1"]
  {:builder-fn rs/as-unqualified-maps})
;; => {:id 1, :name "สมชาย"}

;; Kebab-case keywords
(jdbc/execute-one! ds ["SELECT first_name, last_name FROM users WHERE id = 1"]
  {:builder-fn rs/as-unqualified-kebab-maps})
;; => {:first-name "สม", :last-name "ชาย"}

;; Custom transformation
(defn ->clj [result]
  (map #(update-keys % (fn [k] (keyword (name k)))) result))

;; Reduce with row-fn สำหรับ efficiency
(defn count-users [ds]
  (reduce (fn [acc _] (inc acc))
          0
          (jdbc/plan ds ["SELECT 1 FROM users WHERE active = true"])))

;; Stream large result sets
(defn process-all-users! [ds process-fn]
  (run! process-fn
        (jdbc/plan ds ["SELECT * FROM users ORDER BY id"])))
```

---

## ขั้นตอนที่ 279: Repository Pattern

```clojure
(ns myapp.repository.user
  (:require [next.jdbc :as jdbc]
            [next.jdbc.sql :as sql]
            [honey.sql :as hsql]
            [honey.sql.helpers :as h]))

;; Repository interface
(defprotocol UserRepository
  (find-by-id [this id])
  (find-by-email [this email])
  (find-all [this opts])
  (create! [this user])
  (update! [this id updates])
  (delete! [this id])
  (count-all [this]))

;; PostgreSQL implementation
(defrecord PostgresUserRepository [ds]
  UserRepository
  
  (find-by-id [_ id]
    (when-let [row (sql/get-by-id ds :users id
                     {:builder-fn rs/as-unqualified-kebab-maps})]
      (dissoc row :password-hash)))
  
  (find-by-email [_ email]
    (first (sql/find-by-keys ds :users {:email email}
             {:builder-fn rs/as-unqualified-kebab-maps})))
  
  (find-all [_ {:keys [page size filter-name]}]
    (let [q (cond-> (h/select :*)
              true          (h/from :users)
              filter-name   (h/where [:ilike :name (str "%" filter-name "%")])
              true          (h/order-by [:created-at :desc])
              true          (h/limit (or size 20))
              true          (h/offset (* (dec (or page 1)) (or size 20))))]
      (jdbc/execute! ds (hsql/format q)
        {:builder-fn rs/as-unqualified-kebab-maps})))
  
  (create! [_ user]
    (sql/insert! ds :users
      (select-keys user [:name :email :password-hash :role])
      {:return-keys true
       :builder-fn rs/as-unqualified-kebab-maps}))
  
  (update! [_ id updates]
    (sql/update! ds :users
      (select-keys updates [:name :email :role :active])
      {:id id}
      {:return-keys true
       :builder-fn rs/as-unqualified-kebab-maps}))
  
  (delete! [_ id]
    (sql/delete! ds :users {:id id}))
  
  (count-all [_]
    (:count (jdbc/execute-one! ds ["SELECT COUNT(*) as count FROM users"]))))

;; Factory
(defn create-user-repository [ds]
  (->PostgresUserRepository ds))
```

---

## ขั้นตอนที่ 280: Caching Layer

```clojure
(ns myapp.cache
  (:require [clojure.core.cache :as cache]
            [clojure.core.cache.wrapped :as cw]))

;; TTL cache
(def user-cache
  (cw/ttl-cache-factory {} :ttl 300000))  ; 5 min TTL

(defn get-user-cached [ds user-id]
  (cw/lookup-or-miss user-cache user-id
    (fn [id] (db/find-user ds id))))

(defn invalidate-user! [user-id]
  (swap! user-cache dissoc user-id))

;; LRU cache
(def query-cache
  (cw/lru-cache-factory {} :threshold 1000))  ; max 1000 entries

;; Redis cache (with Carmine)
(require '[taoensso.carmine :as car])

(def redis-conn
  {:pool {} :spec {:uri "redis://localhost:6379"}})

(defmacro redis [& body]
  `(car/wcar redis-conn ~@body))

(defn get-user-redis [user-id]
  (or (some-> (redis (car/get (str "user:" user-id)))
              (clojure.edn/read-string))
      (let [user (db/find-user user-id)]
        (redis (car/setex (str "user:" user-id) 300 (pr-str user)))
        user)))
```

---

## Project Exercise: E-commerce Database Layer

```clojure
(ns ecommerce.db
  (:require [next.jdbc :as jdbc]
            [next.jdbc.sql :as sql]
            [honey.sql :as hsql]
            [honey.sql.helpers :as h]))

;; Schema (SQL)
;; CREATE TABLE products (id, name, price, stock, category_id);
;; CREATE TABLE orders (id, user_id, status, total, created_at);
;; CREATE TABLE order_items (id, order_id, product_id, qty, price);

;; Product queries
(defn get-products-by-category [ds category-id]
  (jdbc/execute! ds
    (hsql/format
      (-> (h/select :*)
          (h/from :products)
          (h/where [:= :category-id category-id] [:> :stock 0])
          (h/order-by :name)))))

;; Create order (transaction)
(defn create-order! [ds user-id items]
  (jdbc/with-transaction [tx ds]
    ;; 1. Verify stock availability
    (doseq [{:keys [product-id qty]} items]
      (let [product (sql/get-by-id tx :products product-id)]
        (when (< (:products/stock product) qty)
          (throw (ex-info "Out of stock"
                          {:product-id product-id
                           :available (:products/stock product)
                           :requested qty})))))
    
    ;; 2. Calculate total
    (let [total (reduce (fn [sum {:keys [product-id qty]}]
                          (let [product (sql/get-by-id tx :products product-id)]
                            (+ sum (* (:products/price product) qty))))
                        0M items)]
      
      ;; 3. Create order
      (let [order (sql/insert! tx :orders
                    {:user_id user-id
                     :status "pending"
                     :total total
                     :created_at (java.time.Instant/now)}
                    {:return-keys true})]
        
        ;; 4. Create order items + decrement stock
        (doseq [{:keys [product-id qty]} items]
          (let [product (sql/get-by-id tx :products product-id)]
            (sql/insert! tx :order_items
              {:order_id (:orders/id order)
               :product_id product-id
               :qty qty
               :price (:products/price product)})
            (sql/update! tx :products
              {:stock (- (:products/stock product) qty)}
              {:id product-id})))
        
        order))))

;; Order summary report
(defn get-order-summary [ds order-id]
  (let [order (sql/get-by-id ds :orders order-id)
        items (jdbc/execute! ds
                ["SELECT oi.*, p.name as product_name
                  FROM order_items oi
                  JOIN products p ON p.id = oi.product_id
                  WHERE oi.order_id = ?"
                 order-id])]
    (assoc order :items items)))

;; REPL testing
(comment
  (def ds (create-pool config))
  
  (create-order! ds 1 [{:product-id 1 :qty 2}
                        {:product-id 3 :qty 1}])
  
  (get-order-summary ds 1))
```

---

### สรุป Database Layer

```
next.jdbc:
  jdbc/execute!      - multiple rows
  jdbc/execute-one!  - single row  
  jdbc/plan          - lazy/streaming
  jdbc/with-transaction - atomic operations
  sql/insert!        - INSERT
  sql/get-by-id      - SELECT by PK
  sql/find-by-keys   - SELECT with WHERE
  sql/update!        - UPDATE
  sql/delete!        - DELETE

HoneySQL:
  (h/select :field1 :field2)
  (h/from :table)
  (h/where [:= :field val])
  (h/join :other [:= :this.id :other.id])
  (h/order-by [:field :desc])
  (h/limit n)
  (h/offset n)
  (hsql/format query) ; → [sql-string & params]

Best Practices:
  ✓ Use parameterized queries (? placeholders)
  ✓ Connection pooling with HikariCP
  ✓ Migrations with Flyway
  ✓ Repository pattern for testability
  ✓ Transactions for multi-step operations
  ✗ Never concatenate user input into SQL!
```

---

*Part 10 จาก 100+ | ขั้นตอน 271-300 จาก 1000+*
