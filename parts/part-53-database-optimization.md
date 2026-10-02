# Part 53: Database Optimization
## ขั้นตอนที่ 1561-1590: Indexes, Query Planning, Connection Pooling, Caching Strategies

---

## บทนำ

Database optimization สำหรับ production:
- **Indexes** - B-tree, GIN, partial indexes
- **Query planning** - EXPLAIN ANALYZE
- **Connection pooling** - HikariCP, PgBouncer
- **N+1 prevention** - batch loading, joins
- **Read replicas** - เพิ่ม read throughput

---

## ขั้นตอนที่ 1561: Index Design

```clojure
;; SQL migrations สำหรับ indexes

;; B-tree index: สำหรับ equality และ range queries
;; CREATE INDEX idx_orders_user_id ON orders(user_id);
;; CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

;; Composite index: ลำดับสำคัญมาก!
;; Most selective column first
;; CREATE INDEX idx_orders_user_status ON orders(user_id, status);
;; ใช้ได้กับ: WHERE user_id = ? AND status = ?
;; ใช้ได้กับ: WHERE user_id = ?
;; ไม่ได้ใช้:  WHERE status = ? เท่านั้น

;; GIN index: สำหรับ JSON/array queries
;; CREATE INDEX idx_products_attrs ON products USING GIN(attributes);
;; ใช้กับ: WHERE attributes @> '{"color": "red"}'

;; Partial index: เฉพาะ active rows
;; CREATE INDEX idx_active_orders ON orders(user_id) WHERE status = 'active';

;; Expression index
;; CREATE INDEX idx_users_lower_email ON users(LOWER(email));
;; ใช้กับ: WHERE LOWER(email) = LOWER(?)

(ns myapp.migrations
  (:require [next.jdbc :as jdbc]))

(defn create-indexes! [db]
  (jdbc/execute! db ["
    -- Orders lookup
    CREATE INDEX CONCURRENTLY IF NOT EXISTS
      idx_orders_user_status_created
      ON orders(user_id, status, created_at DESC);

    -- Full text search
    CREATE INDEX CONCURRENTLY IF NOT EXISTS
      idx_products_fts
      ON products USING GIN(to_tsvector('english', name || ' ' || description));

    -- JSONB attributes
    CREATE INDEX CONCURRENTLY IF NOT EXISTS
      idx_products_attributes
      ON products USING GIN(attributes jsonb_path_ops);

    -- Covering index (includes needed columns to avoid table lookup)
    CREATE INDEX CONCURRENTLY IF NOT EXISTS
      idx_order_items_covering
      ON order_items(order_id)
      INCLUDE (product_id, quantity, price);
  "]))
```

---

## ขั้นตอนที่ 1562: Query Planning ด้วย EXPLAIN

```clojure
(ns myapp.query-analysis
  (:require [next.jdbc :as jdbc]
            [clojure.pprint :refer [pprint]]))

;; Analyze query performance
(defn explain-analyze [db query params]
  (let [explain-sql (str "EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) " query)
        result (jdbc/execute-one! db (into [explain-sql] params))]
    (get result (keyword "QUERY PLAN"))))

;; Example analysis
(defn analyze-order-query [db user-id]
  (let [plan (explain-analyze db
               "SELECT o.*, u.name as user_name
                FROM orders o
                JOIN users u ON o.user_id = u.id
                WHERE o.user_id = ?
                AND o.status = 'active'
                ORDER BY o.created_at DESC
                LIMIT 20"
               [user-id])]
    (println "Execution time:" (get-in plan [0 "Execution Time"]) "ms")
    (println "Planning time:"  (get-in plan [0 "Planning Time"]) "ms")
    plan))

;; Slow query detection
(defn slow-queries [db threshold-ms]
  (jdbc/execute! db
    ["SELECT query, calls, mean_exec_time, stddev_exec_time,
             total_exec_time, rows
      FROM pg_stat_statements
      WHERE mean_exec_time > ?
      ORDER BY mean_exec_time DESC
      LIMIT 20"
     threshold-ms]))

;; Index usage stats
(defn index-usage-stats [db]
  (jdbc/execute! db
    ["SELECT schemaname, tablename, indexname,
             idx_scan, idx_tup_read, idx_tup_fetch
      FROM pg_stat_user_indexes
      ORDER BY idx_scan DESC"]))

;; Unused indexes
(defn unused-indexes [db]
  (jdbc/execute! db
    ["SELECT schemaname, tablename, indexname, pg_size_pretty(pg_relation_size(indexrelid))
      FROM pg_stat_user_indexes
      WHERE idx_scan = 0
      AND NOT indisprimary
      ORDER BY pg_relation_size(indexrelid) DESC"]))
```

---

## ขั้นตอนที่ 1563: Connection Pooling ด้วย HikariCP

```clojure
(ns myapp.database
  (:require [next.jdbc :as jdbc]
            [next.jdbc.connection :as connection])
  (:import [com.zaxxer.hikari HikariDataSource HikariConfig]))

(defn create-connection-pool
  [{:keys [jdbc-url username password
           pool-size min-idle max-lifetime
           connection-timeout idle-timeout]
    :or {pool-size         20
         min-idle          5
         max-lifetime      1800000  ; 30 min
         connection-timeout 30000   ; 30 sec
         idle-timeout       600000}} ; 10 min
   ]
  (let [config (doto (HikariConfig.)
                 (.setJdbcUrl jdbc-url)
                 (.setUsername username)
                 (.setPassword password)
                 (.setMaximumPoolSize pool-size)
                 (.setMinimumIdle min-idle)
                 (.setMaxLifetime max-lifetime)
                 (.setConnectionTimeout connection-timeout)
                 (.setIdleTimeout idle-timeout)
                 ;; PostgreSQL-specific optimizations
                 (.addDataSourceProperty "cachePrepStmts" "true")
                 (.addDataSourceProperty "prepStmtCacheSize" "250")
                 (.addDataSourceProperty "prepStmtCacheSqlLimit" "2048")
                 (.addDataSourceProperty "useServerPrepStmts" "true")
                 (.setPoolName "app-pool"))]
    (HikariDataSource. config)))

;; Monitor pool health
(defn pool-stats [^HikariDataSource pool]
  (let [pool-mx (.getHikariPoolMXBean pool)]
    {:active     (.getActiveConnections pool-mx)
     :idle       (.getIdleConnections pool-mx)
     :waiting    (.getThreadsAwaitingConnection pool-mx)
     :total      (.getTotalConnections pool-mx)}))

;; Integrant lifecycle
(defmethod ig/init-key :app/db [_ config]
  (create-connection-pool config))

(defmethod ig/halt-key! :app/db [_ ^HikariDataSource pool]
  (.close pool))
```

---

## ขั้นตอนที่ 1564: Preventing N+1 Queries

```clojure
(ns myapp.batch-loading
  (:require [next.jdbc :as jdbc]
            [next.jdbc.sql :as sql]))

;; N+1 problem example (BAD)
(defn get-orders-bad [db user-id]
  (let [orders (jdbc/execute! db ["SELECT * FROM orders WHERE user_id = ?" user-id])]
    (map (fn [order]
           ;; This executes 1 query per order!
           (assoc order :items
             (jdbc/execute! db
               ["SELECT * FROM order_items WHERE order_id = ?" (:orders/id order)])))
         orders)))

;; Solution 1: JOIN (for simple cases)
(defn get-orders-with-items-join [db user-id]
  (jdbc/execute! db
    ["SELECT o.id, o.status, o.total,
             oi.id as item_id, oi.product_id, oi.quantity, oi.price
      FROM orders o
      LEFT JOIN order_items oi ON oi.order_id = o.id
      WHERE o.user_id = ?
      ORDER BY o.created_at DESC"
     user-id]))

;; Solution 2: Batch load in 2 queries
(defn get-orders-with-items [db user-id]
  (let [orders     (jdbc/execute! db
                    ["SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC"
                     user-id])
        order-ids  (map :orders/id orders)
        items      (when (seq order-ids)
                     (jdbc/execute! db
                       [(str "SELECT * FROM order_items WHERE order_id IN ("
                              (clojure.string/join "," (repeat (count order-ids) "?"))
                              ")")
                        order-ids]))
        items-by-order (group-by :order_items/order_id items)]
    
    (map (fn [order]
           (assoc order :items (get items-by-order (:orders/id order) [])))
         orders)))

;; Solution 3: DataLoader pattern for GraphQL
(defn create-dataloader [db batch-fn]
  (let [pending (atom [])
        result-p (atom nil)]
    {:load (fn [id]
             (swap! pending conj id)
             (swap! result-p #(or % (delay (batch-fn db @pending))))
             (fn [] (get @@result-p id)))}))
```

---

## ขั้นตอนที่ 1565: Query Optimization Patterns

```clojure
(ns myapp.query-patterns
  (:require [honey.sql :as sql]
            [next.jdbc :as jdbc]))

;; Pagination: offset vs cursor-based
;; Offset pagination: simple but slow for large offsets
(defn paginate-offset [db table page page-size]
  (jdbc/execute! db
    (sql/format {:select   [:*]
                 :from     [table]
                 :order-by [[:id :asc]]
                 :limit    page-size
                 :offset   (* page page-size)})))

;; Cursor pagination: fast, consistent
(defn paginate-cursor [db table cursor page-size]
  (jdbc/execute! db
    (sql/format (cond-> {:select   [:*]
                          :from     [table]
                          :order-by [[:id :asc]]
                          :limit    (inc page-size)}  ; +1 to detect has-more
                  cursor (assoc :where [:> :id cursor])))))

(defn paginate-cursor-result [rows page-size]
  (let [has-more? (> (count rows) page-size)
        items     (take page-size rows)]
    {:items    items
     :has-more has-more?
     :cursor   (when has-more? (:id (last items)))}))

;; Optimistic locking with version
(defn update-with-lock! [db entity-id expected-version updates]
  (let [result (jdbc/execute-one! db
                 (sql/format {:update :orders
                               :set    (assoc updates :version [:+ :version 1])
                               :where  [:and
                                         [:= :id entity-id]
                                         [:= :version expected-version]]
                               :returning [:*]}))]
    (when-not result
      (throw (ex-info "Optimistic lock conflict"
                       {:id entity-id :version expected-version})))))

;; Upsert pattern
(defn upsert-product! [db product]
  (jdbc/execute-one! db
    (sql/format {:insert-into :products
                 :values      [product]
                 :on-conflict [:sku]
                 :do-update-set {:name        :excluded.name
                                  :price       :excluded.price
                                  :updated-at  :%now}})))
```

---

## ขั้นตอนที่ 1566: Read Replicas

```clojure
(ns myapp.replica-routing
  (:require [next.jdbc :as jdbc]))

;; Multiple datasources
(defn create-db-cluster
  [{:keys [primary replicas]}]
  (let [primary-ds    (create-connection-pool primary)
        replica-dss   (mapv create-connection-pool replicas)
        replica-idx   (atom 0)]
    
    {:primary primary-ds
     :replicas replica-dss
     :next-replica
     (fn []
       (when (seq replica-dss)
         (let [idx (mod (swap! replica-idx inc) (count replica-dss))]
           (nth replica-dss idx))))}))

;; Route reads to replicas, writes to primary
(defmacro with-replica [db & body]
  `(let [ds# (or ((:next-replica ~db)) (:primary ~db))]
     (binding [*db* ds#]
       ~@body)))

(defmacro with-primary [db & body]
  `(binding [*db* (:primary ~db)]
     ~@body))

;; Usage
(defn get-products [db query]
  (with-replica db
    (jdbc/execute! *db* query)))

(defn create-product! [db product]
  (with-primary db
    (jdbc/execute-one! *db*
      ["INSERT INTO products ... RETURNING *"])))

;; Automatic routing by operation
(defn execute! [db query]
  (let [sql (first query)
        ds  (if (re-matches #"(?i)^\s*(SELECT|WITH)\s.*" sql)
              (or ((:next-replica db)) (:primary db))
              (:primary db))]
    (jdbc/execute! ds query)))
```

---

## Project: Database Performance Monitor

```clojure
(ns myapp.db-monitor
  (:require [next.jdbc :as jdbc]
            [clojure.core.async :as async]))

(defn collect-db-metrics! [db metrics-store]
  (let [stats {:pool-stats       (pool-stats (:pool db))
               :slow-queries     (slow-queries db 100)
               :table-stats      (jdbc/execute! db
                                   ["SELECT relname, n_live_tup, n_dead_tup,
                                            last_vacuum, last_analyze
                                     FROM pg_stat_user_tables
                                     ORDER BY n_live_tup DESC"])
               :lock-waits       (jdbc/execute! db
                                   ["SELECT count(*) as count
                                     FROM pg_locks l
                                     JOIN pg_stat_activity a ON l.pid = a.pid
                                     WHERE NOT l.granted"])
               :connection-count (jdbc/execute-one! db
                                   ["SELECT count(*) as count FROM pg_stat_activity"])}]
    (swap! metrics-store assoc :db stats)
    stats))

;; Schedule metrics collection
(defn start-monitoring! [db interval-ms]
  (let [metrics (atom {})
        stop-ch (async/chan)]
    (async/go-loop []
      (async/alt!
        (async/timeout interval-ms) ([_]
                                      (try
                                        (collect-db-metrics! db metrics)
                                        (catch Exception e
                                          (println "Metrics error:" (.getMessage e))))
                                      (recur))
        stop-ch ([_] (println "DB monitoring stopped"))))
    {:metrics metrics
     :stop!   #(async/put! stop-ch :stop)}))
```

---

*Part 53 จาก 100+ | ขั้นตอน 1561-1590 จาก 1000+*
