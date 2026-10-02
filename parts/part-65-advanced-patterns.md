# Part 65: Advanced Patterns ระดับโลก
## ขั้นตอนที่ 1921-1950: DSL Design, Macro Systems, Code Generation, Meta-programming

---

## บทนำ

Advanced patterns ของ Clojure experts:
- **DSL (Domain-Specific Languages)** - สร้างภาษาเฉพาะกิจ
- **Macro systems** - compile-time code transformation
- **Code as data** - ใช้ homoiconicity
- **Zipper** - tree manipulation
- **Logic programming** - core.logic

---

## ขั้นตอนที่ 1921: DSL Design Patterns

```clojure
(ns myapp.dsl)

;; Internal DSL สำหรับ workflow definition
;; ใช้ data structures แทน macros ที่ซับซ้อน

;; Workflow definition DSL
(def checkout-workflow
  {:name   "checkout"
   :steps  [{:id      :validate-cart
              :fn      validate-cart
              :timeout 5000
              :on-fail :abort}
             {:id       :check-inventory
              :fn       check-inventory
              :parallel true
              :on-fail  :abort}
             {:id      :calculate-pricing
              :fn      calculate-pricing
              :depends [:validate-cart]}
             {:id      :process-payment
              :fn      process-payment
              :timeout 30000
              :retry   {:max 3 :backoff 1000}
              :on-fail :compensate}
             {:id      :create-order
              :fn      create-order
              :depends [:process-payment :check-inventory]}
             {:id       :send-confirmation
              :fn       send-confirmation
              :async    true
              :on-fail  :continue}]})

;; Workflow executor
(defn execute-workflow! [workflow context]
  (let [steps      (:steps workflow)
        step-map   (into {} (map #(vector (:id %) %) steps))
        completed  (atom {})
        
        execute-step!
        (fn execute-step! [step-id]
          (when-not (contains? @completed step-id)
            (let [step (get step-map step-id)]
              ;; Execute dependencies first
              (doseq [dep (:depends step [])]
                (execute-step! dep))
              
              ;; Execute this step
              (let [result
                    (try
                      (if (:parallel step)
                        (future ((:fn step) context @completed))
                        ((:fn step) context @completed))
                      (catch Exception e
                        (case (:on-fail step)
                          :abort     (throw e)
                          :compensate (do (compensate! @completed) (throw e))
                          :continue  {:error (.getMessage e)}
                          (throw e))))]
                (swap! completed assoc step-id result)))))]
    
    (doseq [step steps]
      (execute-step! (:id step)))
    
    @completed))
```

---

## ขั้นตอนที่ 1922: Macro ขั้นสูง

```clojure
(ns myapp.macros)

;; Timing macro สำหรับ profiling
(defmacro with-timing [name & body]
  `(let [start#  (System/nanoTime)
          result# (do ~@body)
          elapsed# (/ (- (System/nanoTime) start#) 1e6)]
     (metrics/record! :operation-timing
       {:name    ~name
        :elapsed elapsed#
        :success true})
     result#))

;; Tracing macro
(defmacro with-trace [span-name attrs & body]
  `(let [span# (opentelemetry/start-span ~span-name ~attrs)]
     (try
       (let [result# (do ~@body)]
         (opentelemetry/end-span span# :ok)
         result#)
       (catch Exception e#
         (opentelemetry/end-span span# :error e#)
         (throw e#)))))

;; Retry macro with backoff
(defmacro with-retry
  [{:keys [max-attempts backoff-ms jitter?]
    :or   {max-attempts 3 backoff-ms 100 jitter? true}}
   & body]
  `(loop [attempt# 1]
     (let [result# (try
                     {:ok (do ~@body)}
                     (catch Exception e#
                       {:error e#}))]
       (if (:ok result#)
         (:ok result#)
         (if (>= attempt# ~max-attempts)
           (throw (:error result#))
           (let [delay# (* ~backoff-ms (Math/pow 2 (dec attempt#)))
                  jitter# (if ~jitter? (rand-int 100) 0)]
             (Thread/sleep (long (+ delay# jitter#)))
             (recur (inc attempt#))))))))

;; cond-> threading with macros
(defmacro cond->> [expr & clauses]
  (assert (even? (count clauses)))
  (let [pairs (partition 2 clauses)]
    (reduce (fn [acc [test form]]
               `(let [e# ~acc]
                  (if ~test (~form e#) e#)))
             expr
             pairs)))

;; Usage
(cond->> (get-orders db)
  filter-active?   (filter :active?)
  sort-by-date?    (sort-by :created-at >)
  include-details? (map add-order-details))
```

---

## ขั้นตอนที่ 1923: Zipper สำหรับ Tree Manipulation

```clojure
(ns myapp.zipper
  (:require [clojure.zip :as zip]))

;; Zipper: navigate and edit tree structures

;; Hiccup transformation
(defn transform-hiccup [hiccup transform-fn]
  (loop [loc (zip/vector-zip hiccup)]
    (if (zip/end? loc)
      (zip/root loc)
      (let [node (zip/node loc)]
        (recur (zip/next (if (vector? node)
                           (zip/replace loc (transform-fn node))
                           loc)))))))

;; Add CSS class to all divs
(defn add-class-to-divs [class-name]
  (fn [node]
    (if (= :div (first node))
      (let [attrs (if (map? (second node)) (second node) {})]
        (into [:div (update attrs :class
                     #(str % " " class-name))]
               (if (map? (second node))
                 (drop 2 node)
                 (rest node))))
      node)))

;; AST transformation for code analysis
(defn find-all-in-ast [pred ast]
  (loop [loc (zip/seq-zip ast)
          results []]
    (if (zip/end? loc)
      results
      (let [node (zip/node loc)
            new-results (if (pred node)
                           (conj results node)
                           results)]
        (recur (zip/next loc) new-results)))))

;; Example: find all function definitions
(defn find-defns [code-form]
  (find-all-in-ast
    (fn [form]
      (and (seq? form)
           (= 'defn (first form))))
    code-form))
```

---

## ขั้นตอนที่ 1924: Logic Programming ด้วย core.logic

```clojure
(ns myapp.logic
  (:require [clojure.core.logic :as l]
            [clojure.core.logic.pldb :as pldb]))

;; Facts database
(pldb/db-rel parent p c)

(def family-db
  (pldb/db
    [parent :alice :bob]
    [parent :alice :carol]
    [parent :bob   :dave]
    [parent :bob   :eve]))

;; Query: find all children of alice
(pldb/with-db family-db
  (l/run* [q]
    (parent :alice q)))
;; => (:bob :carol)

;; Derived relation: grandparent
(defn grandparent [gp gc]
  (l/fresh [parent-of-gc]
    (parent gp parent-of-gc)
    (parent parent-of-gc gc)))

(pldb/with-db family-db
  (l/run* [q]
    (grandparent :alice q)))
;; => (:dave :eve)

;; Constraint solving
(defn solve-sudoku [puzzle]
  (l/run* [board]
    ;; Constraints
    (l/== (count board) 9)
    ;; Each row has 1-9 with no repeats
    (l/everyg (fn [row]
                (l/permuteo row (range 1 10)))
              board)
    ;; Columns and boxes also...
    ))

;; Type inference example
(defn infer-type [expr env]
  (l/run 1 [type]
    (type-ofo expr env type)))
```

---

## ขั้นตอนที่ 1925: Code Generation

```clojure
(ns myapp.codegen)

;; Generate Clojure code from specifications
(defn generate-crud-namespace [entity-name fields]
  (let [ns-name     (str "myapp." entity-name)
        table-name  (str entity-name "s")
        id-field    (keyword entity-name "id")]
    
    `(ns ~(symbol ns-name)
       (:require [next.jdbc :as jdbc]))
     
     (defn ~(symbol (str "get-" entity-name)) [db id#]
       (jdbc/execute-one! db
         [(str "SELECT * FROM " ~table-name " WHERE id = ?") id#]))
     
     (defn ~(symbol (str "list-" entity-name "s")) [db]
       (jdbc/execute! db
         [(str "SELECT * FROM " ~table-name " ORDER BY id")]))
     
     (defn ~(symbol (str "create-" entity-name "!")) [db data#]
       (jdbc/execute-one! db
         (str "INSERT INTO " ~table-name
              " (" (clojure.string/join ", " (map name ~fields)) ")"
              " VALUES (" (clojure.string/join ", " (repeat (count ~fields) "?")) ")"
              " RETURNING *")
         (map #(get data# %) ~fields)))
     
     (defn ~(symbol (str "update-" entity-name "!")) [db id# data#]
       (jdbc/execute-one! db
         (into [(str "UPDATE " ~table-name
                     " SET " (clojure.string/join ", "
                               (map #(str (name %) " = ?") ~fields))
                     " WHERE id = ? RETURNING *")]
               (conj (mapv #(get data# %) ~fields) id#))))
     
     (defn ~(symbol (str "delete-" entity-name "!")) [db id#]
       (jdbc/execute-one! db
         [(str "DELETE FROM " ~table-name " WHERE id = ? RETURNING *") id#]))))

;; Eval generated code
(eval (generate-crud-namespace "product" [:name :price :sku :stock]))

;; Now we have:
;; (get-product db "123")
;; (list-products db)
;; (create-product! db {:name "Widget" :price 9.99 :sku "W001" :stock 100})
```

---

## Project: Domain-Specific Query Language

```clojure
(ns myapp.query-dsl
  (:require [honey.sql :as sql]))

;; Mini query DSL
;; (query from :products
;;        where (and (= :category "electronics")
;;                   (> :price 100))
;;        select [:id :name :price]
;;        order-by [:price :asc]
;;        limit 20)

(defmacro query [& clauses]
  (let [clause-map (into {} (partition 2 clauses))
        from       (:from clause-map)
        where      (:where clause-map)
        select     (:select clause-map [:*])
        order-by   (:order-by clause-map)
        limit      (:limit clause-map)]
    
    `(sql/format
       ~(cond-> {:select select :from [from]}
          where    (assoc :where (transform-where-clause where))
          order-by (assoc :order-by [(vec order-by)])
          limit    (assoc :limit limit)))))

;; Transform WHERE clause from DSL to HoneySQL
(defmulti transform-where-clause
  (fn [clause] (when (seq? clause) (first clause))))

(defmethod transform-where-clause 'and [clause]
  (into [:and] (map transform-where-clause (rest clause))))

(defmethod transform-where-clause '= [[_ field value]]
  [:= (keyword (name field)) value])

(defmethod transform-where-clause '> [[_ field value]]
  [:> (keyword (name field)) value])

;; Usage
(query from :products
       where (and (= :category "electronics")
                   (> :price 100))
       select [:id :name :price]
       order-by [:price :asc]
       limit 20)
;; => ["SELECT id, name, price FROM products
;;     WHERE category = ? AND price > ?
;;     ORDER BY price ASC LIMIT 20"
;;    "electronics" 100]
```

---

*Part 65 จาก 100+ | ขั้นตอน 1921-1950 จาก 1000+*
