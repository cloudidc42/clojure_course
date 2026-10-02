# Part 99: Clojure Internals & Deep Dive
## ขั้นตอนที่ 2941-2970: Compiler Internals, Persistent Data Structures, Seq Protocol

---

## บทนำ

เข้าใจ Clojure จากภายใน:
- **Persistent data structures** - HAMT, finger trees
- **Seq protocol** - how lazy sequences work
- **Compiler pipeline** - read → analyze → emit
- **STM implementation** - MVCC under the hood
- **Var internals** - namespaces and bindings

---

## ขั้นตอนที่ 2941: Persistent Data Structures

```clojure
;; Clojure's persistent vector: HAMT (Hash Array Mapped Trie)
;; 32-ary tree with path copying for immutability

;; Understand structural sharing
(def v1 (vec (range 1000)))
(def v2 (conj v1 1000))  ; Shares 99%+ of structure with v1

;; PersistentVector node: array of 32 children
;; Each level adds 5 bits (log2(32)) of indexing
;; Level 0 (root): bits 30-25
;; Level 1: bits 24-20
;; ...
;; Level N (leaves): actual values

;; Accessing element: O(log32 N) ≈ O(1) in practice
;; Updating element: O(log32 N) new nodes created

;; Hash map: PersistentHashMap
;; Also HAMT but keyed by hash
;; Perfect for equality-based lookup

;; Understanding hash collisions
(hash "hello")    ; 99162322
(hash :hello)     ; 99162322 (same! by design)
;; => Clojure keys by type+value, never just hash

;; Sorted map: Red-Black tree
(def sm (sorted-map :c 3 :a 1 :b 2))
(keys sm)  ; => (:a :b :c)  -- sorted!

;; Queue: persistent FIFO
(def q (clojure.lang.PersistentQueue/EMPTY))
(def q2 (conj q 1 2 3))
(peek q2)  ; => 1
(pop q2)   ; => queue with 2 3

;; Implement your own persistent structure
(deftype ImmutableStack [elements]
  clojure.lang.IPersistentStack
  (peek [_] (first elements))
  (pop  [_] (ImmutableStack. (rest elements)))
  (cons [_ x] (ImmutableStack. (cons x elements)))
  
  clojure.lang.IPersistentCollection
  (count  [_] (count elements))
  (empty  [_] (ImmutableStack. '()))
  (equiv  [_ other] (= elements (.elements other)))
  (seq    [_] (seq elements))
  
  Object
  (toString [_] (str "Stack" elements)))

(defn make-stack [] (->ImmutableStack '()))
```

---

## ขั้นตอนที่ 2942: Lazy Sequences จากภายใน

```clojure
;; How LazySeq works: thunks and realization

;; LazySeq holds a function that returns nil or an ISeq
;; Once realized, caches the result

;; Understand lazy-seq macro
(defn my-range [start end]
  (lazy-seq
    (when (< start end)
      (cons start (my-range (inc start) end)))))

;; Expansion of (lazy-seq ...)
;; => (clojure.lang.LazySeq. (fn [] ...))

;; LazySeq realization:
;; 1. First time seq/first/rest called
;; 2. Calls the thunk function
;; 3. Caches result
;; 4. Subsequent calls use cache

;; Chunked sequences: 32-element chunks
;; Normal lazy-seq: one element at a time
;; Chunked seq: 32 elements per thunk

(defn chunked-range [n]
  ((fn step [i]
     (when (< i n)
       (let [chunk-size (min 32 (- n i))
             chunk (chunk-buffer chunk-size)]
         (dotimes [j chunk-size]
           (chunk-append chunk (+ i j)))
         (chunk-cons (chunk-first chunk) (step (+ i chunk-size))))))
   0))

;; Force chunked evaluation
(chunked-range 100)  ; Realized 32 at a time

;; Avoiding chunking surprises
(take 1 (map println (range 100)))
;; Prints 32 times due to chunking!

;; To avoid: wrap in a non-chunked lazy-seq
(take 1 (map println (sequence (map identity) (range 100))))
;; Prints exactly 1 time
```

---

## ขั้นตอนที่ 2943: STM Implementation

```clojure
;; STM: Software Transactional Memory
;; Uses MVCC (Multi-Version Concurrency Control)

;; Each Ref has:
;; - Current value
;; - Transaction history (for retry detection)
;; - Fine-grained lock (for writing)
;; - Read point tracking

;; Transaction lifecycle:
;; 1. Transaction starts: records current global clock
;; 2. Reads: checks cached value or reads current
;; 3. Writes: buffer changes in tx-local map
;; 4. Commit attempt:
;;    a. Lock all refs in write set (sorted to avoid deadlock)
;;    b. Verify read set hasn't changed since tx started
;;    c. Write all changes atomically
;;    d. Increment global clock
;;    e. Release locks
;; 5. If verify fails: retry from step 1

;; Commute optimization:
;; commute marks a write as "order-independent"
;; On conflict: re-applies commute function to latest value
;; No retry needed!

;; Why dosync retries are safe:
;; - Transactions are pure functions
;; - Retrying produces same result (eventually)
;; - Side effects in transactions are BAD (run multiple times!)

;; Watch: observe changes to refs
(def counter (ref 0))

(add-watch counter :log
  (fn [key ref old-val new-val]
    (println "Counter changed from" old-val "to" new-val)))

(dosync (alter counter inc))  ; Triggers watch

;; IO transactions: run in transaction, side effects at end
(defn transfer-with-log! [from to amount]
  (dosync
    (alter from - amount)
    (alter to + amount)
    ;; DON'T call send-email! here - it runs on retry too!
    )
  ;; Do side effects AFTER dosync
  (send-transfer-notification! @from @to amount))
```

---

## ขั้นตอนที่ 2944: Var Internals & Namespaces

```clojure
;; Understanding Vars and namespaces

;; Namespace = {symbol -> Var}
;; Var = mutable reference to a value

;; Interning: creating/finding Var in ns
(intern 'user 'x 42)     ; Creates or updates x in user ns
(ns-resolve 'user 'x)    ; Find var by name

;; Var metadata
(def ^{:doc "My function" :deprecated true} old-fn
  (fn [x] (* x 2)))

(meta #'old-fn)
;; => {:doc "My function" :deprecated true :name old-fn :ns #object[...]}

;; Dynamic binding stack
(def ^:dynamic *request-context* nil)

(defn handle-request [req]
  (binding [*request-context* req]
    (process-request)))

;; Binding creates a thread-local stack:
;; thread-A: [*req* = req-A]
;; thread-B: [*req* = req-B]
;; Each thread sees its own value

;; set!: modify current binding (not root value)
(defn log-request! []
  (set! *request-context*
    (assoc *request-context* :logged? true)))

;; Alter-var-root: change root value (affects all threads!)
(alter-var-root #'my-fn (fn [old-fn] new-fn))

;; Namespace introspection
(ns-publics 'clojure.core)    ; All public vars
(ns-imports 'user)            ; Java classes
(ns-refers 'user)             ; Vars referred from other ns
(ns-aliases 'user)            ; Namespace aliases
```

---

## ขั้นตอนที่ 2945: JVM Bytecode Generation

```clojure
;; Clojure compiles directly to JVM bytecode
;; Understanding this helps write performant code

;; Each (defn ...) generates a Java class
;; Class name: namespace$function_name (with mangling)
;; Example: myapp.core/my-fn -> myapp.core$my_fn

;; Functions implement clojure.lang.IFn
;; (invoke ...) is the actual call
;; Fixed arities: invoke(Object arg1, Object arg2, ...)
;; Variadic: applyTo(ISeq args)

;; Closures: captured vars become constructor parameters
(defn make-adder [n]
  (fn [x] (+ n x)))
;; Compiles to: class make_adder__anon__1 implements IFn
;;   field: n
;;   constructor(Object n): this.n = n
;;   invoke(Object x): return n + x

;; Type hints: generate CHECKCAST + direct method call
(defn typed-fn [^String s]
  (.length s))
;; Without hint: reflective call (slow)
;; With hint: INVOKEVIRTUAL java/lang/String.length()I (fast)

;; Primitive arithmetic: avoid boxing
(defn int-sum [^long a ^long b]
  (+ a b))
;; Boxing: creates Long objects for heap
;; Primitive hint: uses JVM longs directly

;; Check compiled bytecode
(require '[clojure.tools.emitter.jvm :as emitter])
;; Or use javap on .class files:
;; javap -c target/classes/myapp/core$my_fn.class
```

---

*Part 99 จาก 100 | ขั้นตอน 2941-2970 จาก 3000*
