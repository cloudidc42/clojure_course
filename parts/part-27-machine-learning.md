# Part 27: Machine Learning ด้วย Clojure
## ขั้นตอนที่ 781-810: Cortex, Neanderthal, DJL, Statistical ML

---

## บทนำ

Machine Learning ใน Clojure ecosystem:
- **Neanderthal** - High-performance matrix/vector operations (BLAS/LAPACK)
- **Deep Diamond** - Neural networks on top of Neanderthal
- **DJL (Deep Java Library)** - JVM ML library ที่ใช้ PyTorch/TensorFlow backend
- **Weka** - Classic ML algorithms ผ่าน JVM
- **Tribuo** - Oracle's ML library สำหรับ JVM
- **clj-ml** - Clojure wrapper สำหรับ ML libraries

---

## ขั้นตอนที่ 781: Linear Algebra ด้วย Neanderthal

```clojure
;; deps.edn
;; {:deps {uncomplicate/neanderthal {:mvn/version "0.47.0"}
;;         uncomplicate/commons {:mvn/version "0.14.0"}}}

(ns ml.linear-algebra
  (:require [uncomplicate.neanderthal.core :refer :all]
            [uncomplicate.neanderthal.native :refer :all]))

;; Vectors
(def v1 (dv [1.0 2.0 3.0]))   ; double vector
(def v2 (dv [4.0 5.0 6.0]))

;; Vector operations
(dot v1 v2)      ; dot product: 1*4 + 2*5 + 3*6 = 32.0
(nrm2 v1)        ; L2 norm: √(1²+2²+3²) = 3.742
(axpy 2.0 v1 v2) ; 2*v1 + v2 = [6.0 9.0 12.0]

;; Matrices
(def m1 (dge 2 3 [1 2 3 4 5 6]))  ; 2x3 matrix
(def m2 (dge 3 2 [7 8 9 10 11 12])) ; 3x2 matrix

(mm m1 m2)  ; Matrix multiply → 2x2 matrix

;; Matrix decompositions
(require '[uncomplicate.neanderthal.linalg :refer :all])

;; LU decomposition
(def a (dge 3 3 [1 2 3 4 5 6 7 8 10]))
(def lu-result (trf a))  ; LU factorization
(trs lu-result (dge 3 1 [1 2 3]))  ; Solve Ax = b

;; SVD
(def svd-result (svd a))
(:sigma svd-result)  ; singular values
```

---

## ขั้นตอนที่ 782: Statistics สำหรับ ML

```clojure
(ns ml.statistics)

;; Feature scaling: Normalization (0-1)
(defn min-max-normalize [data]
  (let [features (apply mapv vector data)
        min-vals  (map #(apply min %) features)
        max-vals  (map #(apply max %) features)]
    (mapv (fn [row]
            (mapv (fn [val mn mx]
                    (if (= mn mx)
                      0.0
                      (/ (- val mn) (- mx mn))))
                  row min-vals max-vals))
          data)))

;; Feature scaling: Standardization (z-score)
(defn standardize [data]
  (let [features  (apply mapv vector data)
        means     (map #(/ (reduce + %) (count %)) features)
        std-devs  (map (fn [feat mean]
                         (let [variance (/ (reduce + (map #(Math/pow (- % mean) 2) feat))
                                           (count feat))]
                           (Math/sqrt variance)))
                       features means)]
    (mapv (fn [row]
            (mapv (fn [val mean std]
                    (if (zero? std)
                      0.0
                      (/ (- val mean) std)))
                  row means std-devs))
          data)))

;; Correlation matrix
(defn pearson-correlation [xs ys]
  (let [n    (count xs)
        mx   (/ (reduce + xs) n)
        my   (/ (reduce + ys) n)
        num  (reduce + (map #(* (- %1 mx) (- %2 my)) xs ys))
        dx   (Math/sqrt (reduce + (map #(Math/pow (- % mx) 2) xs)))
        dy   (Math/sqrt (reduce + (map #(Math/pow (- % my) 2) ys)))]
    (if (or (zero? dx) (zero? dy))
      0.0
      (/ num (* dx dy)))))

;; Train/test split
(defn train-test-split [data test-ratio seed]
  (let [shuffled (shuffle data)
        n        (count data)
        test-n   (int (* n test-ratio))]
    {:train (drop test-n shuffled)
     :test  (take test-n shuffled)}))
```

---

## ขั้นตอนที่ 783: Linear Regression

```clojure
(ns ml.linear-regression)

;; Simple Linear Regression: y = mx + b
;; Ordinary Least Squares (OLS)

(defn linear-regression [xs ys]
  (let [n      (count xs)
        sum-x  (reduce + xs)
        sum-y  (reduce + ys)
        sum-xy (reduce + (map * xs ys))
        sum-x2 (reduce + (map #(* % %) xs))
        
        slope     (/ (- (* n sum-xy) (* sum-x sum-y))
                      (- (* n sum-x2) (* sum-x sum-x)))
        intercept (/ (- sum-y (* slope sum-x)) n)]
    
    {:slope     slope
     :intercept intercept
     :predict   (fn [x] (+ intercept (* slope x)))}))

;; Metrics
(defn rmse [actual predicted]
  (let [errors (map - actual predicted)
        mse    (/ (reduce + (map #(* % %) errors)) (count errors))]
    (Math/sqrt mse)))

(defn r-squared [actual predicted]
  (let [mean-actual (/ (reduce + actual) (count actual))
        ss-total    (reduce + (map #(Math/pow (- % mean-actual) 2) actual))
        ss-residual (reduce + (map #(Math/pow (- %1 %2) 2) actual predicted))]
    (- 1.0 (/ ss-residual ss-total))))

;; Multiple Linear Regression ด้วย Neanderthal
;; X = feature matrix, y = target vector
;; β = (XᵀX)⁻¹Xᵀy

(defn multiple-linear-regression [X y]
  (require '[uncomplicate.neanderthal.core :refer :all]
           '[uncomplicate.neanderthal.linalg :refer :all])
  (let [Xt   (trans X)
       XtX  (mm Xt X)
       Xty  (mv Xt y)
       beta (sv XtX Xty)]  ; solve XtX * β = Xty
    beta))
```

---

## ขั้นตอนที่ 784: Logistic Regression

```clojure
(ns ml.logistic-regression)

;; Sigmoid function
(defn sigmoid [z]
  (/ 1.0 (+ 1.0 (Math/exp (- z)))))

;; Hypothesis function: h(x) = sigmoid(θᵀx)
(defn hypothesis [theta x]
  (sigmoid (reduce + (map * theta x))))

;; Cost function (Binary Cross-Entropy)
(defn cost [theta X y]
  (let [n (count y)
        predictions (map #(hypothesis theta %) X)
        losses (map (fn [h yi]
                      (+ (* yi (Math/log h))
                         (* (- 1 yi) (Math/log (- 1 h)))))
                    predictions y)]
    (- (/ (reduce + losses) n))))

;; Gradient Descent
(defn gradient-descent [X y learning-rate iterations]
  (let [n          (count y)
        m          (count (first X))
        theta-init (vec (repeat m 0.0))]
    
    (loop [theta theta-init
           i     0]
      (if (>= i iterations)
        theta
        (let [predictions (map #(hypothesis theta %) X)
              errors      (map - predictions y)
              gradients   (mapv (fn [j]
                                   (/ (reduce + (map #(* %1 (nth %2 j))
                                                     errors X))
                                      n))
                                (range m))
              new-theta   (mapv - theta (map #(* learning-rate %) gradients))]
          (recur new-theta (inc i)))))))

;; Train and predict
(defn train-logistic [X y]
  (let [theta (gradient-descent X y 0.1 1000)]
    {:theta   theta
     :predict (fn [x] (if (> (hypothesis theta x) 0.5) 1 0))
     :predict-proba (fn [x] (hypothesis theta x))}))
```

---

## ขั้นตอนที่ 785: Decision Tree

```clojure
(ns ml.decision-tree)

;; Gini impurity
(defn gini [labels]
  (let [n      (count labels)
        counts (frequencies labels)]
    (- 1.0 (reduce + (map #(Math/pow (/ % n) 2) (vals counts))))))

;; Information gain
(defn information-gain [parent left right]
  (let [n  (count parent)
        nl (count left)
        nr (count right)]
    (- (gini parent)
       (+ (* (/ nl n) (gini left))
          (* (/ nr n) (gini right))))))

;; Find best split
(defn best-split [X y]
  (let [n-features (count (first X))]
    (reduce
      (fn [best feature-idx]
        (let [values    (sort (set (map #(nth % feature-idx) X)))
              thresholds (partition 2 1 values)]
          (reduce
            (fn [best [a b]]
              (let [threshold (/ (+ a b) 2.0)
                    [left-y right-y] (->> (map vector (map #(nth % feature-idx) X) y)
                                          (partition-by #(< (first %) threshold))
                                          (map #(map second %)))
                    gain (information-gain y (vec left-y) (vec right-y))]
                (if (> gain (:gain best 0))
                  {:feature feature-idx :threshold threshold :gain gain}
                  best)))
            best thresholds)))
      {}
      (range n-features))))

;; Build tree recursively
(defn build-tree [X y max-depth depth]
  (let [majority-class (first (apply max-key val (frequencies y)))]
    (if (or (>= depth max-depth)
            (<= (count (set y)) 1))
      {:leaf true :class majority-class}
      (let [{:keys [feature threshold]} (best-split X y)
            [left-mask right-mask]      (partition-by #(< (nth % feature) threshold) X)
            left-y  (take (count left-mask) y)
            right-y (drop (count left-mask) y)]
        {:leaf       false
         :feature    feature
         :threshold  threshold
         :left       (build-tree (vec left-mask)  (vec left-y)  max-depth (inc depth))
         :right      (build-tree (vec right-mask) (vec right-y) max-depth (inc depth))}))))

;; Predict
(defn predict-tree [tree x]
  (if (:leaf tree)
    (:class tree)
    (if (< (nth x (:feature tree)) (:threshold tree))
      (predict-tree (:left tree) x)
      (predict-tree (:right tree) x))))
```

---

## ขั้นตอนที่ 786: K-Means Clustering

```clojure
(ns ml.kmeans)

;; Euclidean distance
(defn euclidean-distance [a b]
  (Math/sqrt (reduce + (map #(Math/pow (- %1 %2) 2) a b))))

;; Find nearest centroid
(defn nearest-centroid [point centroids]
  (apply min-key #(euclidean-distance point %) centroids))

;; Compute new centroids
(defn new-centroids [clusters]
  (map (fn [cluster]
         (mapv #(/ (reduce + %) (count cluster))
               (apply mapv vector cluster)))
       clusters))

;; K-Means algorithm
(defn kmeans [data k max-iterations]
  (let [initial-centroids (take k (shuffle data))]
    (loop [centroids initial-centroids
           i         0]
      (let [;; Assign each point to nearest centroid
            assignments (map #(nearest-centroid % centroids) data)
            
            ;; Group by centroid
            clusters    (vals (group-by identity
                                         (map vector assignments data)))
            
            ;; Compute new centroids
            new-cents   (new-centroids (map #(map second %) clusters))]
        
        (if (or (= i max-iterations)
                (= centroids new-cents))
          {:centroids new-cents
           :assignments assignments
           :clusters clusters}
          (recur new-cents (inc i)))))))

;; Elbow method to find optimal K
(defn inertia [data assignments centroids]
  (reduce +
    (map #(Math/pow (euclidean-distance %1 %2) 2)
         data assignments)))

(defn find-optimal-k [data max-k]
  (map (fn [k]
         (let [result (kmeans data k 100)]
           {:k        k
            :inertia  (inertia data (:assignments result) (:centroids result))}))
       (range 2 (inc max-k))))
```

---

## ขั้นตอนที่ 787: Using DJL for Deep Learning

```clojure
;; deps.edn
;; {:deps {ai.djl/api {:mvn/version "0.26.0"}
;;         ai.djl.pytorch/pytorch-engine {:mvn/version "0.26.0"}}}

(ns ml.deep-learning
  (:import [ai.djl Model]
           [ai.djl.nn SequentialBlock Blocks]
           [ai.djl.nn.core Linear]
           [ai.djl.training.loss Loss]
           [ai.djl.training DefaultTrainingConfig]
           [ai.djl.training.evaluator Accuracy]
           [ai.djl.training.optimizer Optimizer]
           [ai.djl.ndarray.types Shape]
           [ai.djl.basicdataset.cv.classification ImageFolder]))

;; Build neural network
(defn build-model []
  (let [block (SequentialBlock.)]
    (.add block (Linear/builder)
          (.setUnits 128)
          .build)
    (.add block Blocks/batchFlattenBlock)
    (.add block (Activation/reluBlock))
    (.add block (Linear/builder)
          (.setUnits 64)
          .build)
    (.add block (Activation/reluBlock))
    (.add block (Linear/builder)
          (.setUnits 10)  ; output classes
          .build)
    block))

;; Train model
(defn train-model! [train-dataset test-dataset epochs]
  (with-open [model (Model/newInstance "mnist")]
    (.setBlock model (build-model))
    
    (let [config (-> (DefaultTrainingConfig. (Loss/softmaxCrossEntropyLoss))
                     (.addEvaluator (Accuracy.))
                     (.optOptimizer
                       (-> (Optimizer/adam)
                           (.optLearningRateTracker
                             (LrScheduler/fixed 0.001))
                           .build)))]
      
      (with-open [trainer (.newTrainer model config)]
        (.initialize trainer (into-array Shape [(Shape. (long-array [1 784]))]))
        
        (dotimes [epoch epochs]
          (EasyTrain/trainEpoch trainer train-dataset)
          (EasyTrain/evaluateDataset trainer test-dataset)
          (println (format "Epoch %d: accuracy=%.4f"
                           (inc epoch)
                           (-> trainer .getTrainingResult .getValidateEvaluation
                               (get "Accuracy")))))
        
        (.save model (java.nio.file.Paths/get "model" (into-array String [])) "mnist")))))
```

---

## Project: Text Classification (Sentiment Analysis)

```clojure
(ns ml.sentiment
  (:require [clojure.string :as str]))

;; Naive Bayes สำหรับ Text Classification

;; Tokenize text
(defn tokenize [text]
  (-> text
      str/lower-case
      (str/replace #"[^\w\s]" "")
      (str/split #"\s+")))

;; Build vocabulary
(defn build-vocab [texts]
  (set (mapcat tokenize texts)))

;; Count word frequencies per class
(defn build-model [training-data]
  (let [classes    (set (map :label training-data))
        by-class   (group-by :label training-data)
        class-probs (into {} (map (fn [[cls items]]
                                    [cls (/ (count items) (count training-data))])
                                  by-class))
        word-counts (into {}
                      (map (fn [[cls items]]
                              [cls (frequencies (mapcat (comp tokenize :text) items))])
                           by-class))
        vocab-size  (count (set (mapcat (comp tokenize :text) training-data)))]
    {:class-probs class-probs
     :word-counts word-counts
     :vocab-size  vocab-size}))

;; Predict class using Naive Bayes
(defn predict [model text]
  (let [{:keys [class-probs word-counts vocab-size]} model
        tokens (tokenize text)]
    (apply max-key val
      (into {}
        (map (fn [[cls prior]]
               (let [wc (get word-counts cls {})
                     total (reduce + (vals wc))
                     log-likelihood (reduce +
                                     (map (fn [token]
                                            ;; Laplace smoothing
                                            (Math/log (/ (inc (get wc token 0))
                                                         (+ total vocab-size))))
                                          tokens))]
                 [cls (+ (Math/log prior) log-likelihood)]))
             class-probs)))))

;; Train and evaluate
(def training-data
  [{:text "สินค้าดีมากครับ ชอบมาก" :label :positive}
   {:text "คุณภาพยอดเยี่ยม ราคาเหมาะสม" :label :positive}
   {:text "แย่มาก ไม่คุ้มค่าเลย" :label :negative}
   {:text "ผิดหวังมาก คุณภาพต่ำ" :label :negative}])

(def model (build-model training-data))

(predict model "สินค้าดีมาก แนะนำเลย")
;; => :positive
```

---

*Part 27 จาก 100+ | ขั้นตอน 781-810 จาก 1000+*
