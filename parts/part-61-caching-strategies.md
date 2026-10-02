# Part 61: Caching Strategies
## ขั้นตอนที่ 1801-1830: Redis Caching, Cache-Aside, Write-Through, Cache Invalidation

---

## บทนำ

Caching ระดับ production:
- **Cache-aside** - application controls cache
- **Write-through** - write to cache and DB together
- **Read-through** - cache loads on miss
- **Cache invalidation** - เมื่อ data เปลี่ยน
- **Distributed caching** - Redis Cluster

---

## ขั้นตอนที่ 1801: Redis Caching Basics

```clojure
(ns myapp.cache
  (:require [taoensso.carmine :as car]
            [cheshire.core :as json]))

(def redis-pool {:pool {} :spec {:host "localhost" :port 6379}})

(defmacro wcar [& body]
  `(car/wcar redis-pool ~@body))

;; Basic cache operations
(defn cache-get [key]
  (when-let [data (wcar (car/get key))]
    (json/parse-string data true)))

(defn cache-set! [key value ttl-seconds]
  (wcar (car/setex key ttl-seconds (json/generate-string value)))
  value)

(defn cache-del! [key]
  (wcar (car/del key)))

(defn cache-exists? [key]
  (= 1 (wcar (car/exists key))))

;; Memoize with Redis
(defn memoize-redis [f key-fn ttl-seconds]
  (fn [& args]
    (let [cache-key (apply key-fn args)]
      (or (cache-get cache-key)
          (let [result (apply f args)]
            (cache-set! cache-key result ttl-seconds)
            result)))))

;; Usage
(def get-product-cached
  (memoize-redis
    get-product-from-db
    (fn [id] (str "product:" id))
    3600))  ; 1 hour TTL
```

---

## ขั้นตอนที่ 1802: Cache-Aside Pattern

```clojure
(ns myapp.cache.cache-aside
  (:require [myapp.cache :as cache]
            [myapp.db :as db]))

;; Read: check cache first, fallback to DB
(defn get-user [db user-id]
  (let [cache-key (str "user:" user-id)]
    (or (cache/cache-get cache-key)
        (when-let [user (db/get-user-by-id db user-id)]
          (cache/cache-set! cache-key user 1800)  ; 30 min
          user))))

;; Write: update DB then invalidate cache
(defn update-user! [db user-id updates]
  (let [updated (db/update-user! db user-id updates)]
    (cache/cache-del! (str "user:" user-id))
    updated))

;; Stale-while-revalidate pattern
(defn get-with-stale [cache-key fetch-fn ttl stale-ttl]
  (let [cached (cache/cache-get cache-key)]
    (cond
      ;; Fresh cache
      (and cached (< (:cached-at cached 0)
                      (+ (System/currentTimeMillis) (* ttl 1000))))
      (:value cached)
      
      ;; Stale: return stale data and refresh in background
      (and cached (< (:cached-at cached 0)
                      (+ (System/currentTimeMillis) (* stale-ttl 1000))))
      (do
        (future
          (let [fresh (fetch-fn)]
            (cache/cache-set! cache-key
              {:value fresh :cached-at (System/currentTimeMillis)}
              stale-ttl)))
        (:value cached))
      
      ;; Cache miss: fetch synchronously
      :else
      (let [fresh (fetch-fn)]
        (cache/cache-set! cache-key
          {:value fresh :cached-at (System/currentTimeMillis)}
          stale-ttl)
        fresh))))
```

---

## ขั้นตอนที่ 1803: Write-Through Cache

```clojure
;; Write to cache and DB simultaneously
(defn write-through! [db cache key value ttl-seconds]
  (jdbc/with-transaction [tx db]
    (let [db-result (db/upsert! tx key value)]
      (cache/cache-set! (str "entity:" key) value ttl-seconds)
      db-result)))

;; Batch write-through
(defn batch-write-through! [db items key-fn ttl-seconds]
  (jdbc/with-transaction [tx db]
    (let [db-results (mapv #(db/upsert! tx (key-fn %) %) items)]
      (doseq [item items]
        (cache/cache-set! (str "entity:" (key-fn item)) item ttl-seconds))
      db-results)))

;; Write-behind (async): write to cache immediately, DB later
(defonce write-queue (atom []))

(defn write-behind! [key value ttl-seconds]
  (cache/cache-set! key value ttl-seconds)
  (swap! write-queue conj {:key key :value value :at (System/currentTimeMillis)}))

;; Flush queue to DB periodically
(defn start-write-behind-flusher! [db interval-ms]
  (future
    (loop []
      (Thread/sleep interval-ms)
      (when-let [pending (seq @write-queue)]
        (swap! write-queue #(drop (count pending) %))
        (jdbc/with-transaction [tx db]
          (doseq [{:keys [key value]} pending]
            (db/upsert! tx key value))))
      (recur))))
```

---

## ขั้นตอนที่ 1804: Cache Invalidation Strategies

```clojure
;; Event-based cache invalidation
(defn invalidate-on-event! [event]
  (case (:type event)
    :product-updated
    (do
      (cache/cache-del! (str "product:" (:id (:data event))))
      (cache/cache-del! "products:list:*"))  ; wildcard not supported - use tags
    
    :order-placed
    (do
      (cache/cache-del! (str "user:orders:" (:user-id (:data event))))
      (cache/cache-del! (str "product:stock:" (:product-id (:data event)))))
    
    nil))

;; Cache tags for group invalidation
(defn cache-with-tags! [key value ttl tags]
  (cache/cache-set! key value ttl)
  (doseq [tag tags]
    (wcar (car/sadd (str "tag:" tag) key)
           (car/expire (str "tag:" tag) (* ttl 2)))))

(defn invalidate-tag! [tag]
  (let [keys (wcar (car/smembers (str "tag:" tag)))]
    (when (seq keys)
      (wcar (apply car/del keys))
      (wcar (car/del (str "tag:" tag))))))

;; Usage
(defn get-category-products [db category]
  (let [cache-key (str "products:category:" category)]
    (or (cache/cache-get cache-key)
        (let [products (db/get-products-by-category db category)]
          (cache-with-tags! cache-key products 3600
            ["products" (str "category:" category)])
          products))))

;; When any product changes
(defn invalidate-product-caches! [product-id category]
  (cache/cache-del! (str "product:" product-id))
  (invalidate-tag! "products")
  (invalidate-tag! (str "category:" category)))
```

---

## ขั้นตอนที่ 1805: Distributed Cache Patterns

```clojure
;; Two-level cache: L1 = in-memory, L2 = Redis
(defn create-two-level-cache [l2-ttl l1-ttl l1-max-size]
  (let [l1 (atom (java.util.LinkedHashMap. 16 0.75 true))  ; LRU map
        evict-l1! (fn []
                    (when (> (.size @l1) l1-max-size)
                      (.remove @l1 (first (.keySet @l1)))))]
    {:get
     (fn [key]
       ;; Check L1 first
       (or (.get @l1 key)
           ;; L2 (Redis)
           (when-let [v (cache/cache-get key)]
             ;; Populate L1
             (evict-l1!)
             (.put @l1 key v)
             v)))
     
     :set!
     (fn [key value]
       ;; Write to both levels
       (cache/cache-set! key value l2-ttl)
       (evict-l1!)
       (.put @l1 key value)
       value)
     
     :del!
     (fn [key]
       (.remove @l1 key)
       (cache/cache-del! key))}))

;; Distributed lock for cache stampede prevention
(defn with-cache-lock [key ttl-ms f]
  (let [lock-key (str key ":lock")
        acquired (= "OK" (wcar (car/set lock-key 1 "NX" "PX" ttl-ms)))]
    (if acquired
      (try
        (f)
        (finally
          (wcar (car/del lock-key))))
      ;; Another process is computing - wait briefly
      (do
        (Thread/sleep 100)
        (cache/cache-get key)))))
```

---

## Project: Caching Layer สำหรับ E-Commerce

```clojure
(ns myapp.cache.ecommerce
  (:require [myapp.cache :as cache]))

;; Product cache with smart invalidation
(defrecord ProductCache [l2-ttl l1-cache]
  
  clojure.lang.IFn
  (invoke [this product-id]
    (let [cache-key (str "product:" product-id)]
      (or (get-in @l1-cache [cache-key])
          (cache/cache-get cache-key))))
  
  Object
  (toString [_] "ProductCache"))

(defn cached-get-product [db product-id]
  (let [key (str "product:" product-id)]
    (or (cache/cache-get key)
        (when-let [p (db/get-product db product-id)]
          (cache/cache-set! key p 3600)
          p))))

(defn cached-get-recommendations [db user-id]
  (let [key (str "recommendations:" user-id)]
    (or (cache/cache-get key)
        (let [recs (db/compute-recommendations db user-id)]
          (cache/cache-set! key recs 900)  ; 15 min - stale is ok
          recs))))

;; Pre-warm cache on startup
(defn prewarm-cache! [db]
  (let [popular (db/get-popular-products db 100)]
    (doseq [p popular]
      (cache/cache-set! (str "product:" (:id p)) p 3600))
    (println "Pre-warmed" (count popular) "products")))
```

---

*Part 61 จาก 100+ | ขั้นตอน 1801-1830 จาก 1000+*
