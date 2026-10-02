# Part 67: Functional Programming เชิงลึก
## ขั้นตอนที่ 1981-2010: Monads, Functors, Category Theory, Transducers ขั้นสูง

---

## บทนำ

Functional programming concepts ขั้นสูง:
- **Functors** - mappable containers
- **Monads** - sequenceable effects
- **Applicatives** - function application in context
- **Transducers** - composable transformations
- **Free monads** - interpret effects later

---

## ขั้นตอนที่ 1981: Functors

```clojure
(ns myapp.fp)

;; Functor: สิ่งที่สามารถ map ได้
;; fmap : (a -> b) -> f a -> f b

;; Clojure's built-in functors:
;; - List: (map f coll)
;; - Maybe: (some-> x f)
;; - Either: (if-let [v x] (f v) err)

;; Custom Maybe functor
(defrecord Just [value])
(defrecord Nothing [])

(defn just [v] (->Just v))
(defn nothing [] (->Nothing))
(defn just? [m] (instance? Just m))

(defprotocol Functor
  (fmap [this f]))

(extend-protocol Functor
  Just
  (fmap [this f] (just (f (:value this))))
  
  Nothing
  (fmap [this _] this)
  
  clojure.lang.PersistentList
  (fmap [this f] (map f this))
  
  clojure.lang.PersistentVector
  (fmap [this f] (mapv f this))
  
  nil
  (fmap [_ _] nil))

;; Usage
(fmap (just 5) inc)       ; => Just{value: 6}
(fmap (nothing) inc)      ; => Nothing{}
(fmap [1 2 3] #(* % 2))  ; => [2 4 6]

;; Functor laws:
;; 1. Identity: fmap id = id
;; 2. Composition: fmap (f . g) = fmap f . fmap g

(defn verify-functor-laws [functor-val]
  (let [id-law     (= (fmap functor-val identity) functor-val)
        comp-f     #(* % 2)
        comp-g     #(+ % 1)
        comp-law   (= (fmap functor-val (comp comp-f comp-g))
                       (fmap (fmap functor-val comp-g) comp-f))]
    {:identity    id-law
     :composition comp-law
     :valid?      (and id-law comp-law)}))
```

---

## ขั้นตอนที่ 1982: Monads

```clojure
;; Monad: Functor + pure + bind
;; pure : a -> m a
;; bind : m a -> (a -> m b) -> m b  (also >>= in Haskell)

(defprotocol Monad
  (pure   [m v])
  (bind   [m f]))

(extend-protocol Monad
  Just
  (pure [_ v] (just v))
  (bind [this f]
    (if (just? this)
      (f (:value this))
      (nothing)))
  
  Nothing
  (pure [_ _] (nothing))
  (bind [this _] this))

;; Monad comprehension (do-notation equivalent)
(defmacro m-do [bindings & body]
  (if (empty? bindings)
    `(do ~@body)
    (let [[binding-form monad-val & rest-bindings] bindings]
      `(bind ~monad-val
             (fn [~binding-form]
               (m-do ~rest-bindings ~@body))))))

;; Usage: chain operations that might fail
(defn safe-divide [a b]
  (if (zero? b) (nothing) (just (/ a b))))

(m-do [x (just 10)
       y (just 5)
       z (safe-divide x y)]
  (just (* z 3)))
;; => Just{value: 6}

(m-do [x (just 10)
       y (just 0)
       z (safe-divide x y)]  ; fails here
  (just (* z 3)))
;; => Nothing{}

;; List monad (non-determinism)
(extend-protocol Monad
  clojure.lang.PersistentVector
  (pure [_ v] [v])
  (bind [this f] (vec (mapcat f this))))

;; Pythagorean triples using list monad
(m-do [a (range 1 20)
       b (range a 20)
       c (range b 20)
       _ (if (= (+ (* a a) (* b b)) (* c c)) [[]] [])]
  [[a b c]])
```

---

## ขั้นตอนที่ 1983: Applicative Functors

```clojure
;; Applicative: Functor + apply
;; apply : f (a -> b) -> f a -> f b

(defprotocol Applicative
  (ap [this other]))

(extend-protocol Applicative
  Just
  (ap [this other]
    (if (and (just? this) (just? other))
      (just ((:value this) (:value other)))
      (nothing)))
  
  Nothing
  (ap [_ _] (nothing)))

;; Usage
(ap (just inc) (just 5))  ; => Just{value: 6}
(ap (just +)   (just 3))  ; => function waiting for second arg

;; Lift: apply pure function to multiple applicative values
(defn lift-a2 [f fa fb]
  (ap (fmap fa f) fb))

(lift-a2 + (just 3) (just 4))  ; => Just{value: 7}
(lift-a2 + (nothing) (just 4)) ; => Nothing{}

;; Validation applicative (collect all errors)
(defrecord Success [value])
(defrecord Failure [errors])

(defn success [v] (->Success v))
(defn failure [& errors] (->Failure (vec errors)))

(extend-protocol Functor
  Success
  (fmap [this f] (success (f (:value this))))
  Failure
  (fmap [this _] this))

(extend-protocol Applicative
  Success
  (ap [this other]
    (cond
      (instance? Failure this)  this
      (instance? Failure other) other
      :else (success ((:value this) (:value other)))))
  Failure
  (ap [this other]
    (if (instance? Failure other)
      (failure (concat (:errors this) (:errors other)))
      this)))
```

---

## ขั้นตอนที่ 1984: Transducers ขั้นสูง

```clojure
;; Transducers: composable, context-free transformations
;; xf : (result input -> result) -> (result input -> result)

;; Custom transducer: sliding window
(defn sliding-window-xf [n]
  (fn [rf]
    (let [window (atom (java.util.ArrayDeque. n))]
      (fn
        ([]  (rf))
        ([result] (rf result))
        ([result input]
         (.addLast @window input)
         (when (> (.size @window) n)
           (.removeFirst @window))
         (if (= (.size @window) n)
           (rf result (vec @window))
           result))))))

;; Usage
(into [] (sliding-window-xf 3) [1 2 3 4 5])
;; => [[1 2 3] [2 3 4] [3 4 5]]

;; Stateful transducer: moving average
(defn moving-average-xf [n]
  (comp
    (sliding-window-xf n)
    (map #(/ (reduce + %) (double n)))))

(into [] (moving-average-xf 3) [1 2 3 4 5 6])
;; => [2.0 3.0 4.0 5.0]

;; Transducer with side effects (logging)
(defn log-xf [label]
  (fn [rf]
    (fn
      ([] (rf))
      ([result] (rf result))
      ([result input]
       (println label "Processing:" input)
       (rf result input)))))

;; Parallel transducer (using fork-join)
(defn parallel-map-xf [f parallelism]
  (fn [rf]
    (let [pool    (java.util.concurrent.ForkJoinPool. parallelism)
          pending (java.util.concurrent.ConcurrentLinkedQueue.)]
      (fn
        ([] (rf))
        ([result]
         ;; Drain pending futures
         (loop [r result]
           (if-let [fut (.poll pending)]
             (recur (rf r @fut))
             (rf r))))
        ([result input]
         (.add pending (.submit pool (fn [] (f input))))
         (if (> (.size pending) (* 2 parallelism))
           (rf result @(.poll pending))
           result))))))
```

---

## ขั้นตอนที่ 1985: Free Monads

```clojure
;; Free Monad: describe computation without executing it
;; Lets you interpret the same program multiple ways

;; Database algebra
(defrecord GetUser [id])
(defrecord SaveUser [user])
(defrecord DeleteUser [id])
(defrecord FindUsers [query])

;; Free structure
(defrecord Pure  [value])
(defrecord Impure [operation k])

(defn free-pure [v] (->Pure v))

(defn free-bind [m f]
  (if (instance? Pure m)
    (f (:value m))
    (->Impure (:operation m)
               (comp (fn [x] (free-bind x f))
                      (:k m)))))

;; Program description (no side effects yet!)
(defn get-user-program [user-id]
  (->Impure (->GetUser user-id) free-pure))

(defn program []
  (free-bind (get-user-program "123")
    (fn [user]
      (free-bind (get-user-program (:manager-id user))
        (fn [manager]
          (free-pure {:user user :manager manager}))))))

;; Interpreter 1: Real DB
(defn interpret-db [program db]
  (if (instance? Pure program)
    (:value program)
    (let [op     (:operation program)
          result (cond
                    (instance? GetUser op)    (db/get-user db (:id op))
                    (instance? SaveUser op)   (db/save-user! db (:user op))
                    (instance? DeleteUser op) (db/delete-user! db (:id op)))]
      (interpret-db ((:k program) result) db))))

;; Interpreter 2: In-memory for testing
(defn interpret-test [program test-db]
  (if (instance? Pure program)
    (:value program)
    (let [op     (:operation program)
          result (cond
                    (instance? GetUser op) (get test-db (:id op))
                    (instance? SaveUser op) (:user op))]
      (interpret-test ((:k program) result) test-db))))
```

---

## Project: Functional Data Pipeline

```clojure
(ns myapp.fp-pipeline)

;; Compose functional transformations
(defn pipeline [& xfs]
  (apply comp (reverse xfs)))

(defn run-pipeline [data & xfs]
  (transduce (apply pipeline xfs) conj [] data))

;; Usage: ETL pipeline
(def etl-pipeline
  (pipeline
    (filter :active?)
    (map (fn [record]
            (-> record
                (update :name clojure.string/trim)
                (update :email clojure.string/lower-case))))
    (remove #(clojure.string/blank? (:email %)))
    (map #(assoc % :processed-at (java.time.Instant/now)))))

(run-pipeline raw-data etl-pipeline)
```

---

*Part 67 จาก 100+ | ขั้นตอน 1981-2010 จาก 1000+*
