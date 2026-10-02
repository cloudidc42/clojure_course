# Part 25: Redis และ Caching
## ขั้นตอนที่ 721-750: Carmine, Cache Patterns, Pub/Sub, Distributed Lock

---

## บทนำ

Redis เป็น in-memory data structure store ที่ใช้สำหรับ:
- **Caching** - เพิ่ม performance โดย cache ผล database query
- **Session storage** - เก็บ session ผู้ใช้
- **Pub/Sub** - real-time messaging
- **Rate limiting** - จำกัด request rate
- **Distributed locks** - synchronize ระหว่าง instances
- **Leaderboards** - Sorted Sets สำหรับ ranking

ใน Clojure ใช้ library **Carmine** ที่เป็น idiomatic Redis client

---

## ขั้นตอนที่ 721: Setup และ Connection

```clojure
;; deps.edn
;; {:deps {com.taoensso/carmine {:mvn/version "3.3.2"}}}

(ns myapp.redis
  (:require [taoensso.carmine :as car :refer [wcar]]))

;; Connection pool
(def redis-conn
  {:pool {:max-total     8
          :max-idle      4
          :min-idle      2
          :test-on-borrow true}
   :spec {:host     (or (System/getenv "REDIS_HOST") "localhost")
          :port     (Integer/parseInt (or (System/getenv "REDIS_PORT") "6379"))
          :password (System/getenv "REDIS_PASSWORD")
          :db       (Integer/parseInt (or (System/getenv "REDIS_DB") "0"))
          :timeout-ms 3000}})

;; Helper macro (ใช้บ่อยมาก)
(defmacro redis [& body]
  `(wcar redis-conn ~@body))

;; Test connection
(redis (car/ping))  ; => "PONG"
(redis (car/info "server"))
```

---

## ขั้นตอนที่ 722: Basic Operations

```clojure
;; ===== String operations =====

;; SET / GET
(redis (car/set "greeting" "สวัสดี"))
(redis (car/get "greeting"))  ; => "สวัสดี"

;; SET with expiry (seconds)
(redis (car/setex "session:abc123" 3600 "{\"user\":1}"))
(redis (car/ttl "session:abc123"))  ; => 3600 (seconds remaining)

;; GET and SET atomically
(redis (car/getset "counter" "0"))

;; Increment
(redis (car/set "views" 0))
(redis (car/incr "views"))      ; => 1
(redis (car/incr "views"))      ; => 2
(redis (car/incrby "views" 5))  ; => 7
(redis (car/get "views"))       ; => "7"

;; DEL
(redis (car/del "greeting"))

;; EXISTS
(redis (car/exists "views"))  ; => 1 (exists)
(redis (car/exists "nope"))   ; => 0

;; EXPIRE
(redis (car/expire "views" 60))  ; expire in 60 seconds

;; Batch operations (pipeline)
(redis
  (car/set "a" 1)
  (car/set "b" 2)
  (car/set "c" 3))
;; All 3 execute in one network roundtrip!

;; MGET
(redis (car/mget "a" "b" "c"))  ; => ["1" "2" "3"]
```

---

## ขั้นตอนที่ 723: Data Structures - Lists

```clojure
;; ===== Lists =====
;; Use cases: queues, recent items, activity feeds

;; Push to head/tail
(redis (car/lpush "queue" "task3" "task2" "task1"))
(redis (car/rpush "queue" "task4"))

;; Pop from head/tail
(redis (car/lpop "queue"))   ; => "task1" (FIFO queue)
(redis (car/rpop "queue"))   ; => "task4" (LIFO stack)

;; Blocking pop (wait up to 5 seconds)
(redis (car/blpop "queue" 5))

;; View without removing
(redis (car/lrange "queue" 0 -1))  ; all items
(redis (car/lrange "queue" 0 4))   ; first 5
(redis (car/llen "queue"))          ; length

;; Recent items (keep last 100)
(defn add-activity! [user-id action]
  (let [key (str "activity:" user-id)]
    (redis
      (car/lpush key (pr-str {:action action
                               :time   (System/currentTimeMillis)}))
      (car/ltrim key 0 99))))  ; keep only last 100

(defn get-recent-activity [user-id n]
  (->> (redis (car/lrange (str "activity:" user-id) 0 (dec n)))
       (map read-string)))
```

---

## ขั้นตอนที่ 724: Data Structures - Sets and Hashes

```clojure
;; ===== Sets =====
;; Use cases: unique items, tags, followers, blocked users

(redis (car/sadd "post:1:tags" "clojure" "programming" "functional"))
(redis (car/smembers "post:1:tags"))  ; => #{"clojure" "programming" "functional"}
(redis (car/sismember "post:1:tags" "clojure"))  ; => 1 (true)
(redis (car/scard "post:1:tags"))  ; => 3

;; Set operations
(redis (car/sadd "post:2:tags" "clojure" "web" "backend"))
(redis (car/sinter "post:1:tags" "post:2:tags"))  ; intersection: #{"clojure"}
(redis (car/sunion "post:1:tags" "post:2:tags"))  ; union
(redis (car/sdiff "post:1:tags" "post:2:tags"))   ; difference

;; ===== Hashes =====
;; Use cases: objects, user profiles

(redis
  (car/hset "user:100"
    "name"    "สมชาย"
    "email"   "somchai@test.com"
    "age"     "25"
    "city"    "Bangkok"))

(redis (car/hget  "user:100" "name"))  ; => "สมชาย"
(redis (car/hgetall "user:100"))        ; => map of all fields
(redis (car/hmget "user:100" "name" "email"))  ; => ["สมชาย" "somchai@test.com"]

(redis (car/hincrby "user:100" "login_count" 1))
(redis (car/hdel "user:100" "age"))
(redis (car/hexists "user:100" "name"))  ; => 1

;; Store Clojure map in hash
(defn save-user! [user-id user-map]
  (apply redis
    (car/hmset* (str "user:" user-id)
      (into {} (map (fn [[k v]] [(name k) (pr-str v)]) user-map)))))

(defn get-user [user-id]
  (let [raw (redis (car/hgetall (str "user:" user-id)))]
    (into {} (map (fn [[k v]] [(keyword k) (read-string v)]) raw))))
```

---

## ขั้นตอนที่ 725: Sorted Sets (Leaderboards)

```clojure
;; ===== Sorted Sets =====
;; Use cases: leaderboards, rankings, rate limiting

;; Add to sorted set (value + score)
(redis
  (car/zadd "leaderboard" 5000 "player:alice")
  (car/zadd "leaderboard" 3200 "player:bob")
  (car/zadd "leaderboard" 7500 "player:carol")
  (car/zadd "leaderboard" 4100 "player:dave"))

;; Get top 10 (highest score first)
(redis (car/zrevrange "leaderboard" 0 9))
;; => ["player:carol" "player:alice" "player:dave" "player:bob"]

;; With scores
(redis (car/zrevrange "leaderboard" 0 9 "WITHSCORES"))
;; => ["player:carol" 7500 "player:alice" 5000 ...]

;; Get rank (0-indexed, lower = better rank)
(redis (car/zrevrank "leaderboard" "player:alice"))  ; => 1 (2nd place)
(redis (car/zscore   "leaderboard" "player:alice"))  ; => 5000.0

;; Update score
(redis (car/zincrby "leaderboard" 1000 "player:bob"))  ; bob gets +1000

;; Rate limiting ด้วย sorted set
(defn rate-limit! [key limit window-ms]
  (let [now    (System/currentTimeMillis)
        cutoff (- now window-ms)]
    (redis
      ;; Remove old requests
      (car/zremrangebyscore key "-inf" cutoff)
      ;; Add current request
      (car/zadd key now (str now))
      ;; Count recent requests
      (car/zcard key)
      ;; Set expiry
      (car/expire key (int (/ window-ms 1000))))
    (let [count (last (redis (car/zcard key)))]
      (<= count limit))))

;; ใช้งาน: max 100 requests per minute
(rate-limit! "ratelimit:user:1" 100 60000)
```

---

## ขั้นตอนที่ 726: Cache-Aside Pattern

```clojure
;; Pattern ที่ใช้บ่อยที่สุด: Cache-Aside (Lazy Loading)
;; 1. Check cache
;; 2. If miss: fetch from DB, store in cache
;; 3. Return data

(defn cache-aside
  "Generic cache-aside helper"
  [cache-key ttl-seconds fetch-fn]
  (or (redis (car/get cache-key))
      (let [data (fetch-fn)]
        (when data
          (redis (car/setex cache-key ttl-seconds (pr-str data))))
        data)))

;; ใช้งาน
(defn get-user-profile [user-id]
  (cache-aside
    (str "profile:" user-id)
    3600  ; cache for 1 hour
    (fn [] (db/find-user user-id))))

;; Invalidate cache เมื่อข้อมูลเปลี่ยน
(defn update-user-profile! [user-id updates]
  (db/update-user! user-id updates)
  (redis (car/del (str "profile:" user-id))))  ; invalidate!

;; Cache with JSON serialization
(require '[cheshire.core :as json])

(defn json-cache [cache-key ttl-seconds fetch-fn]
  (if-let [cached (redis (car/get cache-key))]
    (json/parse-string cached true)
    (let [data (fetch-fn)]
      (when data
        (redis (car/setex cache-key ttl-seconds (json/generate-string data))))
      data)))

;; Cache with Memoize decorator
(defn memoize-with-redis [f cache-key-fn ttl]
  (fn [& args]
    (let [key (apply cache-key-fn args)]
      (json-cache key ttl #(apply f args)))))

(def get-product
  (memoize-with-redis
    db/find-product
    (fn [id] (str "product:" id))
    300))
```

---

## ขั้นตอนที่ 727: Write-Through Cache

```clojure
;; Write-Through: เขียน cache และ DB พร้อมกัน
;; Pro: Cache always fresh
;; Con: Write latency slightly higher

(defn write-through!
  "Write to both DB and cache atomically"
  [entity-type id data]
  (let [cache-key (str (name entity-type) ":" id)]
    (db/upsert! entity-type id data)
    (redis (car/setex cache-key 3600 (json/generate-string data)))
    data))

;; Product update with write-through
(defn update-product! [product-id updates]
  (let [updated (db/update-product! product-id updates)]
    (redis (car/setex (str "product:" product-id)
                       300
                       (json/generate-string updated)))
    updated))

;; Cache warming (pre-populate cache)
(defn warm-product-cache! []
  (let [popular-products (db/find-popular-products 100)]
    (redis
      (doseq [p popular-products]
        (car/setex (str "product:" (:id p))
                   3600
                   (json/generate-string p))))
    (println "Warmed" (count popular-products) "products")))
```

---

## ขั้นตอนที่ 728: Redis Pub/Sub

```clojure
;; Publish/Subscribe สำหรับ real-time events
;; ใช้ได้ดีสำหรับ:
;; - Cache invalidation across instances
;; - Real-time notifications
;; - Event broadcasting

;; Publisher
(defn publish-event! [channel event]
  (redis (car/publish channel (json/generate-string event))))

;; Subscriber (blocking loop in separate thread)
(defn subscribe-events! [channels handler]
  (future
    (car/with-new-pubsub-listener
      (:spec redis-conn)
      {"message" (fn [msg]
                    (let [[_ channel data] msg]
                      (handler channel (json/parse-string data true))))}
      (apply car/subscribe channels)
      @(promise))))  ; block forever

;; Cache invalidation via Pub/Sub
(def cache-invalidation-listener
  (subscribe-events!
    ["cache:invalidate"]
    (fn [_ {:keys [entity-type id]}]
      (redis (car/del (str (name entity-type) ":" id)))
      (println "Invalidated cache:" entity-type id))))

;; When updating, publish invalidation
(defn update-with-invalidation! [entity-type id data]
  (db/update! entity-type id data)
  (publish-event! "cache:invalidate"
                   {:entity-type (name entity-type) :id id}))
```

---

## ขั้นตอนที่ 729: Distributed Locking

```clojure
;; Distributed Lock ด้วย Redis
;; ใช้ SETNX + EXPIRE (atomic ด้วย SET NX EX)

(defn acquire-lock!
  "Try to acquire distributed lock. Returns lock-value if acquired, nil if not."
  [lock-name ttl-seconds]
  (let [lock-value (str (java.util.UUID/randomUUID))]
    (when (= "OK" (redis (car/set lock-name lock-value
                                   "NX"  ; only set if Not eXists
                                   "EX"  ; Expire
                                   ttl-seconds)))
      lock-value)))

(defn release-lock!
  "Release lock only if we own it (prevents releasing another process's lock)"
  [lock-name lock-value]
  (let [script "if redis.call('get', KEYS[1]) == ARGV[1] then
                  return redis.call('del', KEYS[1])
                else
                  return 0
                end"]
    (= 1 (redis (car/eval script 1 lock-name lock-value)))))

(defmacro with-distributed-lock
  "Execute body while holding distributed lock"
  [lock-name ttl & body]
  `(let [lock-value# (acquire-lock! ~lock-name ~ttl)]
     (if lock-value#
       (try
         ~@body
         (finally
           (release-lock! ~lock-name lock-value#)))
       (throw (ex-info "Could not acquire lock"
                        {:lock ~lock-name})))))

;; ใช้งาน: prevent duplicate payment processing
(defn process-payment! [order-id]
  (with-distributed-lock
    (str "payment:" order-id) 30  ; lock for 30 seconds
    (when-not (db/already-paid? order-id)
      (let [result (payment-gateway/charge! order-id)]
        (db/mark-paid! order-id result)
        result))))
```

---

## ขั้นตอนที่ 730: Session Storage ใน Redis

```clojure
;; Store user sessions in Redis
;; Advantages:
;; - Shared across multiple app instances
;; - Automatic expiry
;; - Fast access

(def session-ttl 86400)  ; 24 hours

(defn create-session! [user-id data]
  (let [session-id (str (java.util.UUID/randomUUID))]
    (redis
      (car/setex (str "session:" session-id)
                  session-ttl
                  (json/generate-string (assoc data :user-id user-id))))
    session-id))

(defn get-session [session-id]
  (when-let [data (redis (car/get (str "session:" session-id)))]
    (json/parse-string data true)))

(defn update-session! [session-id updates]
  (when-let [current (get-session session-id)]
    (let [updated (merge current updates)]
      (redis
        (car/setex (str "session:" session-id)
                    session-ttl
                    (json/generate-string updated)))
      updated)))

(defn destroy-session! [session-id]
  (redis (car/del (str "session:" session-id))))

;; Ring middleware สำหรับ Redis sessions
(defn wrap-redis-session [handler]
  (fn [request]
    (let [session-id (get-in request [:cookies "session" :value])
          session    (when session-id (get-session session-id))
          request'   (assoc request :session (or session {}))]
      (let [response  (handler request')
            new-session (:session response)]
        (if new-session
          (let [sid (or session-id (create-session! nil new-session))]
            (update-session! sid new-session)
            (-> response
                (dissoc :session)
                (assoc-in [:cookies "session"] {:value sid :http-only true})))
          response)))))
```

---

## Project: Shopping Cart ด้วย Redis

```clojure
(ns shop.cart
  (:require [taoensso.carmine :as car :refer [wcar]]
            [cheshire.core :as json]))

;; Cart key = "cart:{user-id}" หรือ "cart:guest:{session-id}"
(defn cart-key [user-id] (str "cart:" user-id))

;; Add item to cart
(defn add-to-cart! [user-id product-id quantity]
  (let [key (cart-key user-id)
        field (str "item:" product-id)]
    (redis
      (car/hincrby key field quantity)
      (car/expire key 86400))))  ; cart expires after 24h

;; Remove item
(defn remove-from-cart! [user-id product-id]
  (redis (car/hdel (cart-key user-id) (str "item:" product-id))))

;; Get cart
(defn get-cart [user-id]
  (let [raw (redis (car/hgetall (cart-key user-id)))]
    (->> (partition 2 raw)
         (map (fn [[k v]]
                [(-> k (clojure.string/replace #"^item:" "") Long/parseLong)
                 (Integer/parseInt v)]))
         (into {}))))

;; Get cart with product details
(defn get-cart-with-details [user-id]
  (let [cart (get-cart user-id)]
    (map (fn [[product-id qty]]
           (let [product (get-product product-id)]
             {:product  product
              :quantity qty
              :subtotal (* (:price product) qty)}))
         cart)))

;; Cart total
(defn cart-total [user-id]
  (->> (get-cart-with-details user-id)
       (map :subtotal)
       (reduce + 0)))

;; Clear cart
(defn clear-cart! [user-id]
  (redis (car/del (cart-key user-id))))

;; Merge guest cart on login
(defn merge-guest-cart! [guest-id user-id]
  (let [guest-cart (get-cart (str "guest:" guest-id))]
    (doseq [[product-id qty] guest-cart]
      (add-to-cart! user-id product-id qty))
    (redis (car/del (cart-key (str "guest:" guest-id))))))
```

---

*Part 25 จาก 100+ | ขั้นตอน 721-750 จาก 1000+*
