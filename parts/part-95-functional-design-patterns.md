# Part 95: Functional Design Patterns ขั้นสูง
## ขั้นตอนที่ 2821-2850: Lenses, Zippers, CPS, Trampolining, Partial Evaluation

---

## บทนำ

Functional patterns ระดับ world-class:
- **Lenses** - composable data access/update
- **Zippers** - navigate and edit tree structures
- **Continuation-Passing Style (CPS)** - explicit control flow
- **Trampolining** - avoid stack overflow in recursion
- **Partial evaluation** - optimize at compile time

---

## ขั้นตอนที่ 2821: Lenses สำหรับ Immutable Data

```clojure
(ns myapp.fp.lenses)

;; A Lens is a pair of [getter setter]
;; Getter: data -> value
;; Setter: data -> new-value -> new-data

(defrecord Lens [get set])

(defn lens [get-fn set-fn]
  (->Lens get-fn set-fn))

;; Primitive lens operations
(defn view [lens data]
  ((:get lens) data))

(defn set-val [lens data value]
  ((:set lens) data value))

(defn over [lens data f]
  (set-val lens data (f (view lens data))))

;; Common lenses
(defn key-lens [k]
  (lens #(get % k)
        #(assoc %1 k %2)))

(def first-lens
  (lens first #(into [%2] (rest %1))))

(defn nth-lens [i]
  (lens #(nth % i)
        #(assoc %1 i %2)))

;; Compose lenses (focus deeper)
(defn compose-lenses [& lenses]
  (reduce (fn [acc l]
            (lens
              (fn [data] (view l (view acc data)))
              (fn [data val]
                (over acc data #(set-val l % val)))))
          (first lenses)
          (rest lenses)))

;; Named compositions
(def city-lens
  (compose-lenses (key-lens :address)
                  (key-lens :city)))

;; Example
(def user {:name "Alice" :address {:city "Bangkok" :postal "10100"}})

(view city-lens user)         ; => "Bangkok"
(set-val city-lens user "Chiang Mai")  ; => {:name "Alice" :address {:city "Chiang Mai" :postal "10100"}}
(over city-lens user clojure.string/upper-case)  ; => {:name "Alice" :address {:city "BANGKOK" ...}}

;; Prism: optional lens (for sum types)
(defn prism [preview review]
  {:preview preview :review review})

(def some-prism
  (prism identity (fn [_ v] v)))

;; Traverse: apply to multiple focus points
(defn traverse [traversal data f]
  (reduce #(over (key-lens %2) %1 f)
          data
          traversal))
```

---

## ขั้นตอนที่ 2822: Zippers สำหรับ Tree Editing

```clojure
(ns myapp.fp.zippers
  (:require [clojure.zip :as z]))

;; Zipper: cursor into a tree structure

;; Navigate AST
(defn find-node [zipper pred]
  (loop [loc zipper]
    (cond
      (z/end? loc) nil
      (pred (z/node loc)) loc
      :else (recur (z/next loc)))))

;; Replace all matching nodes
(defn replace-all [zipper pred transform-fn]
  (loop [loc zipper]
    (if (z/end? loc)
      (z/root loc)
      (recur (z/next
               (if (pred (z/node loc))
                 (z/replace loc (transform-fn (z/node loc)))
                 loc))))))

;; Edit nested config tree
(def config-zip
  (z/zipper map?
             (fn [m] (seq (vals m)))
             (fn [m children]
               (zipmap (keys m) children))
             {:db {:host "localhost" :port 5432}
              :cache {:host "redis" :port 6379}}))

;; Navigate to db config
(-> config-zip
    (z/down)          ; {:host "localhost" :port 5432}
    (z/node))

;; HTML tree manipulation
(defn html-zipper [html-tree]
  (z/zipper
    #(and (vector? %) (not (string? (first %))))
    (fn [node] (filter vector? (drop 2 node)))
    (fn [node children]
      (into (take 2 node) children))
    html-tree))

;; Add class to all div elements
(defn add-class [html class-name]
  (replace-all
    (html-zipper html)
    #(and (vector? %) (= :div (first %)))
    (fn [[tag attrs & children]]
      (into [tag (update attrs :class str " " class-name)]
            children))))
```

---

## ขั้นตอนที่ 2823: Continuation-Passing Style (CPS)

```clojure
;; CPS: explicit continuation (what to do next)

;; Normal style
(defn factorial [n]
  (if (<= n 1) 1 (* n (factorial (- n 1)))))

;; CPS style: passes result to continuation k
(defn factorial-cps [n k]
  (if (<= n 1)
    (k 1)
    (factorial-cps (- n 1)
      (fn [result] (k (* n result))))))

;; Call it
(factorial-cps 10 identity)  ; => 3628800

;; CPS enables:
;; 1. Tail-call optimization
;; 2. Explicit error handling
;; 3. Coroutines / async

;; CPS with error handling
(defn safe-divide-cps [a b success failure]
  (if (zero? b)
    (failure "Division by zero")
    (success (/ a b))))

(defn calculate-cps [x y z success failure]
  (safe-divide-cps x y
    (fn [q1]
      (safe-divide-cps q1 z
        success
        failure))
    failure))

(calculate-cps 100 5 2
  (fn [result] (println "Result:" result))
  (fn [error] (println "Error:" error)))

;; CPS for async composition
(defn fetch-user-cps [user-id callback error-cb]
  (future
    (try
      (callback (fetch-from-db user-id))
      (catch Exception e
        (error-cb (.getMessage e))))))

(defn fetch-orders-cps [user-id callback error-cb]
  (fetch-user-cps user-id
    (fn [user]
      (callback (fetch-orders-for-user user)))
    error-cb))
```

---

## ขั้นตอนที่ 2824: Trampolining สำหรับ Deep Recursion

```clojure
;; Trampoline: avoid stack overflow in mutual recursion

;; Without trampoline: StackOverflowError for large n
(defn even? [n]
  (if (zero? n) true (odd? (dec n))))
(defn odd? [n]
  (if (zero? n) false (even? (dec n))))

;; With trampoline: return thunk instead of calling directly
(defn even-t? [n]
  (if (zero? n) true #(odd-t? (dec n))))
(defn odd-t? [n]
  (if (zero? n) false #(even-t? (dec n))))

(trampoline even-t? 100000)  ; Works!

;; General trampoline
(defn my-trampoline [f & args]
  (loop [result (apply f args)]
    (if (fn? result)
      (recur (result))
      result)))

;; Parse expression with trampolining
(defn parse-expr [tokens]
  (letfn [(parse-primary [tokens]
            (let [[tok & rest] tokens]
              (cond
                (number? tok) [tok rest]
                (= :lparen tok)
                (let [[expr remaining] (trampoline #(parse-expr tokens))]
                  ...))))
          
          (parse-term [tokens]
            (let [[left remaining] (parse-primary tokens)]
              (if (#{:mul :div} (first remaining))
                #(parse-term (rest remaining))
                [left remaining])))]
    (parse-term tokens)))

;; Tree traversal without stack overflow
(defn count-nodes-trampoline [tree]
  (trampoline
    (fn count-t [node acc]
      (if (nil? node)
        acc
        (fn []
          (count-t (:left node)
            (fn []
              (count-t (:right node) (inc acc)))))))
    tree 0))
```

---

## ขั้นตอนที่ 2825: Memoization Strategies

```clojure
;; Advanced memoization patterns

;; Standard memoize
(def fib
  (memoize
    (fn [n]
      (if (< n 2) n
          (+ (fib (- n 1)) (fib (- n 2)))))))

;; Memoize with cache eviction (LRU)
(defn lru-memoize [f max-size]
  (let [cache (java.util.LinkedHashMap. max-size 0.75 true)]
    (fn [& args]
      (locking cache
        (or (.get cache args)
            (let [result (apply f args)]
              (when (>= (.size cache) max-size)
                (.remove cache (-> cache .entrySet .iterator .next .getKey)))
              (.put cache args result)
              result))))))

;; Memoize with dependency tracking
(defn reactive-memoize [compute-fn deps-fn]
  (let [cache (atom {})]
    (fn [& args]
      (let [deps     (deps-fn args)
            cached   (get @cache args)
            valid?   (and cached (= (:deps cached) deps))]
        (if valid?
          (:value cached)
          (let [value (apply compute-fn args)]
            (swap! cache assoc args {:value value :deps deps})
            value))))))

;; Memoize with max-staleness
(defn timed-memoize [f ttl-ms]
  (let [cache (atom {})]
    (fn [& args]
      (let [now    (System/currentTimeMillis)
            cached (get @cache args)]
        (if (and cached (< (- now (:ts cached)) ttl-ms))
          (:value cached)
          (let [value (apply f args)]
            (swap! cache assoc args {:value value :ts now})
            value))))))
```

---

*Part 95 จาก 100+ | ขั้นตอน 2821-2850 จาก 1000+*
