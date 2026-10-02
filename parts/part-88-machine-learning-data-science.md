# Part 88: Machine Learning & Data Science
## ขั้นตอนที่ 2611-2640: Numeric Computing, ML Models, Data Pipelines, Visualization

---

## บทนำ

Data Science ด้วย Clojure:
- **tech.ml.dataset** - DataFrame-like operations
- **Neanderthal** - high-performance linear algebra
- **Libpython-clj** - use Python ML libraries
- **Tableplot/Hanami** - data visualization
- **Nextjournal Clerk** - computational notebooks

---

## ขั้นตอนที่ 2611: Dataset Processing ด้วย tech.ml.dataset

```clojure
(ns myapp.data.dataset
  (:require [tech.v3.dataset :as ds]
            [tech.v3.datatype :as dtype]
            [tech.v3.datatype.functional :as dfn]))

;; Load CSV
(def orders-ds
  (ds/->dataset "orders.csv"
    {:key-fn keyword}))

;; Explore dataset
(ds/head orders-ds 5)
(ds/shape orders-ds)     ; [rows cols]
(ds/column-names orders-ds)
(ds/describe orders-ds)  ; stats for each column

;; Filter rows
(ds/filter orders-ds #(> (:total %) 100))

;; Select columns
(ds/select-columns orders-ds [:order-id :customer-id :total :status])

;; Add computed column
(defn add-discount-column [ds]
  (ds/add-column ds
    (ds/new-column :discounted-total
      (dfn/* (ds :total) 0.9))))

;; Group by and aggregate
(-> orders-ds
    (ds/group-by :status)
    (ds/aggregate {:order-count (ds/row-count)
                    :total-revenue #(dfn/sum (% :total))
                    :avg-total     #(dfn/mean (% :total))}))

;; Join datasets
(defn enrich-orders [orders-ds customers-ds]
  (ds/join-by-column orders-ds customers-ds :customer-id))

;; Sort
(ds/sort-by orders-ds :total >)

;; Pipeline
(defn clean-orders [raw-ds]
  (-> raw-ds
      (ds/filter #(not (nil? (:customer-id %))))
      (ds/update-column :status #(map clojure.string/lower-case %))
      (ds/add-column :total-with-tax
        (ds/new-column :total-with-tax
          (dfn/* (raw-ds :total) 1.07)))
      (ds/drop-columns [:raw-notes :internal-id])))
```

---

## ขั้นตอนที่ 2612: Linear Algebra ด้วย Neanderthal

```clojure
(ns myapp.data.linalg
  (:require [uncomplicate.neanderthal.core :refer :all]
            [uncomplicate.neanderthal.native :refer :all]))

;; Native vectors and matrices (zero-copy, BLAS-backed)
(def v1 (dv [1.0 2.0 3.0]))
(def v2 (dv [4.0 5.0 6.0]))

;; Vector operations
(dot v1 v2)    ; dot product = 32.0
(axpy 2.0 v1 v2)  ; 2*v1 + v2

;; Matrix operations
(def A (dge 2 3 [1 2 3 4 5 6]))  ; 2x3 matrix
(def B (dge 3 2 [7 8 9 10 11 12]))  ; 3x2 matrix

(mm A B)  ; matrix multiply: 2x2 result

;; Solve linear system Ax = b
(defn solve-linear [A b]
  (let [A-copy (copy A)
        b-copy (copy b)]
    (sv A-copy b-copy)
    b-copy))

;; SVD for dimensionality reduction
(defn pca-reduce [data-matrix n-components]
  (let [U S Vt (svd data-matrix)]
    ;; Keep top n_components
    (mm (view-ge U [0 0] [(mrows U) n-components])
        (view-ge (dia S) [0 0] [n-components n-components]))))

;; GPU acceleration with CUDA (if available)
;; (require '[uncomplicate.neanderthal.cuda :refer [cuv]])
;; (def gpu-v (cuv [1.0 2.0 3.0]))
```

---

## ขั้นตอนที่ 2613: ML mit scicloj/sklearn-clj

```clojure
(ns myapp.ml.models
  (:require [tablecloth.api :as tc]
            [scicloj.ml.core :as ml]
            [scicloj.ml.metamorph :as mm]
            [tech.v3.dataset :as ds]))

;; Load and prepare data
(def titanic-ds
  (-> (ds/->dataset "titanic.csv" {:key-fn keyword})
      (tc/select-columns [:survived :pclass :sex :age :fare])
      (tc/drop-missing)))

;; Feature engineering
(def prepared-ds
  (-> titanic-ds
      (tc/add-column :sex-numeric #(map {:male 0 :female 1} (% :sex)))
      (tc/drop-columns [:sex])))

;; Train/test split
(def {:keys [train test]} (tc/split prepared-ds :holdout {:ratio 0.8}))

;; Build pipeline with metamorph
(defn build-pipeline []
  (mm/pipeline
    {:metamorph/id :model}
    (ml/model {:model-type :smile.classification/random-forest
               :n-trees    100})))

;; Train model
(defn train-model [train-ds]
  (let [pipeline (build-pipeline)]
    (ml/train pipeline train-ds {:target-columns [:survived]})))

;; Evaluate model
(defn evaluate [model test-ds]
  (let [predictions (ml/predict model test-ds)
        actual      (vec (test-ds :survived))
        predicted   (vec (:survived predictions))]
    {:accuracy (/ (count (filter identity (map = actual predicted)))
                   (count actual))}))

;; Cross-validation
(defn cross-validate [ds n-folds]
  (let [folds (tc/k-fold-split ds n-folds)]
    (map (fn [{:keys [train test]}]
            (let [model (train-model train)]
              (evaluate model test)))
         folds)))
```

---

## ขั้นตอนที่ 2614: Libpython-clj สำหรับ Python ML

```clojure
(ns myapp.ml.python-bridge
  (:require [libpython-clj2.require :refer [require-python]]
            [libpython-clj2.python :refer [py. py.. py.-] :as py]))

;; Import Python libraries
(require-python '[numpy :as np])
(require-python '[pandas :as pd])
(require-python '[sklearn.ensemble :as ske])
(require-python '[sklearn.model_selection :as skms])
(require-python '[sklearn.preprocessing :as skp])
(require-python '[sklearn.metrics :as skmet])

;; Use sklearn RandomForest
(defn train-sklearn-model [X-train y-train]
  (let [rf (ske/RandomForestClassifier
             :n_estimators 100
             :max_depth 5
             :random_state 42)]
    (py. rf fit X-train y-train)
    rf))

;; Full ML pipeline with sklearn
(defn run-ml-pipeline [data-path target-col]
  (let [df      (pd/read_csv data-path)
        X       (py. df drop [target-col] :axis 1)
        y       (py.- df target-col)
        
        ;; Scale features
        scaler  (skp/StandardScaler)
        X-scaled (py. scaler fit_transform X)
        
        ;; Split
        {:keys [X-train X-test y-train y-test]}
        (let [split (skms/train_test_split X-scaled y :test_size 0.2 :random_state 42)]
          {:X-train (first split)
           :X-test  (second split)
           :y-train (nth split 2)
           :y-test  (nth split 3)})
        
        ;; Train
        model (train-sklearn-model X-train y-train)
        
        ;; Evaluate
        y-pred  (py. model predict X-test)
        score   (skmet/accuracy_score y-test y-pred)]
    
    {:model    model
     :accuracy score
     :report   (skmet/classification_report y-test y-pred)}))

;; Convert between Clojure and NumPy
(defn clj->numpy [coll]
  (np/array coll))

(defn numpy->clj [arr]
  (vec (py. arr tolist)))
```

---

## ขั้นตอนที่ 2615: Data Visualization ด้วย Hanami

```clojure
(ns myapp.viz.hanami
  (:require [aerial.hanami.common :as hc]
            [aerial.hanami.templates :as ht]
            [aerial.hanami.core :as hmi]))

;; Start visualization server
(hmi/start-server 3333)

;; Simple line chart
(defn time-series-chart [data title x-key y-key]
  (hc/xform ht/line-chart
    :DATA     data
    :X        (name x-key)
    :Y        (name y-key)
    :TITLE    title
    :WIDTH    800
    :HEIGHT   300))

;; Revenue over time
(defn revenue-chart [db]
  (let [data (revenue-by-day db "2024-01-01" "2024-12-31")]
    (time-series-chart
      (map (fn [row]
               {:date (str (:day row))
                :revenue (double (:revenue row))})
           data)
      "Daily Revenue"
      :date :revenue)))

;; Bar chart for status distribution
(defn order-status-chart [db]
  (let [data (orders-by-status db)]
    (hc/xform ht/bar-chart
      :DATA  (map (fn [r] {:status (name (:status r))
                             :count  (:count r)}) data)
      :X     "status"
      :Y     "count"
      :TITLE "Orders by Status")))

;; Scatter plot for price vs demand
(defn price-demand-scatter [products]
  (hc/xform ht/point-chart
    :DATA  (map #(select-keys % [:price :monthly-sales :category]) products)
    :X     "price"
    :Y     "monthly-sales"
    :COLOR {:field "category" :type "nominal"}
    :TITLE "Price vs Demand"))

;; Dashboard: multiple charts
(defn dashboard [db]
  (hmi/sv! :revenue-dashboard
    (hc/xform ht/vconcat-chart
      :VCONCAT [(revenue-chart db)
                (order-status-chart db)])))
```

---

*Part 88 จาก 100+ | ขั้นตอน 2611-2640 จาก 1000+*
