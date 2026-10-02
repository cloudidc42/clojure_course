# Part 63: Machine Learning กับ Clojure
## ขั้นตอนที่ 1861-1890: Linear Regression, Neural Networks, Recommendations, NLP

---

## บทนำ

ML ใน Clojure:
- **Fastmath** - numerical computing
- **Smile** - machine learning library  
- **Deep Learning 4J** - neural networks
- **Recommendations** - collaborative filtering
- **NLP** - text processing

---

## ขั้นตอนที่ 1861: Numerical Computing ด้วย Fastmath

```clojure
(ns myapp.ml
  (:require [fastmath.core :as m]
            [fastmath.stats :as stats]))

;; Basic statistics
(def data [1.0 2.0 3.0 4.0 5.0 6.0 7.0 8.0 9.0 10.0])

(stats/mean data)           ; => 5.5
(stats/median data)         ; => 5.5
(stats/stddev data)         ; => 3.03
(stats/percentile data 95)  ; => 9.55

;; Correlation
(def x [1 2 3 4 5])
(def y [2 4 6 8 10])
(stats/pearson-r x y)  ; => 1.0 (perfect correlation)

;; Linear regression (manual implementation)
(defn linear-regression [xs ys]
  (let [n     (count xs)
        x-mean (stats/mean xs)
        y-mean (stats/mean ys)
        
        slope (/ (reduce + (map #(* (- %1 x-mean) (- %2 y-mean)) xs ys))
                  (reduce + (map #(m/sq (- % x-mean)) xs)))
        
        intercept (- y-mean (* slope x-mean))]
    
    {:slope     slope
     :intercept intercept
     :predict   (fn [x] (+ intercept (* slope x)))
     :r-squared (let [y-pred    (map #(+ intercept (* slope %)) xs)
                       ss-res    (reduce + (map #(m/sq (- %1 %2)) ys y-pred))
                       ss-tot    (reduce + (map #(m/sq (- % y-mean)) ys))]
                   (- 1 (/ ss-res ss-tot)))}))

;; Usage
(def model (linear-regression
             [1 2 3 4 5 6 7 8 9 10]
             [1.1 2.3 2.9 4.1 5.0 5.9 7.1 8.0 9.2 10.1]))

(println "Slope:" (:slope model))
(println "R²:" (:r-squared model))
((:predict model) 11)  ; Predict for x=11
```

---

## ขั้นตอนที่ 1862: K-Means Clustering

```clojure
(defn euclidean-distance [p1 p2]
  (m/sqrt (reduce + (map #(m/sq (- %1 %2)) p1 p2))))

(defn nearest-centroid [point centroids]
  (apply min-key #(euclidean-distance point %) centroids))

(defn update-centroid [points]
  (let [n (count points)]
    (mapv #(/ (reduce + (map % points)) n)
           (range (count (first points))))))

(defn k-means [data k max-iter]
  (loop [centroids (take k (shuffle data))
          iter      0]
    
    (let [clusters     (group-by #(nearest-centroid % centroids) data)
          new-centroids (mapv (fn [c]
                                (update-centroid (get clusters c [c])))
                               centroids)]
      
      (if (or (= iter max-iter)
               (= centroids new-centroids))
        {:centroids new-centroids
         :clusters  (vals clusters)
         :iters     iter}
        (recur new-centroids (inc iter))))))

;; Usage: cluster customers by purchase behavior
(def customer-data
  [[100 5]    ; [avg_order_value, frequency]
   [200 3]
   [50  10]
   [150 4]
   [80  8]
   [300 2]])

(def result (k-means customer-data 3 100))
(println "Clusters:" (count (:clusters result)))
(println "Centroids:" (:centroids result))
```

---

## ขั้นตอนที่ 1863: Collaborative Filtering

```clojure
;; User-item matrix based recommendations
(defn cosine-similarity [v1 v2]
  (let [dot-product (reduce + (map * v1 v2))
        magnitude   (* (m/sqrt (reduce + (map m/sq v1)))
                        (m/sqrt (reduce + (map m/sq v2))))]
    (if (zero? magnitude) 0.0
        (/ dot-product magnitude))))

(defn user-similarity [ratings user1 user2]
  (let [common-items (for [[item r1] (get ratings user1)
                             :when (contains? (get ratings user2) item)]
                       [r1 (get-in ratings [user2 item])])
        
        v1 (map first common-items)
        v2 (map second common-items)]
    
    (if (< (count common-items) 2)
      0.0
      (cosine-similarity v1 v2))))

;; Find similar users
(defn find-similar-users [ratings user-id n]
  (->> (keys ratings)
       (filter #(not= % user-id))
       (map (fn [other-user]
               {:user       other-user
                :similarity (user-similarity ratings user-id other-user)}))
       (sort-by :similarity >)
       (take n)))

;; Predict rating using similar users
(defn predict-rating [ratings user-id item-id n-neighbors]
  (let [similar-users (find-similar-users ratings user-id n-neighbors)
        
        ;; Filter to users who rated this item
        relevant-users (filter #(contains? (get ratings (:user %)) item-id)
                                 similar-users)
        
        weighted-sum  (reduce + (map #(* (:similarity %)
                                          (get-in ratings [(:user %) item-id]))
                                       relevant-users))
        
        total-weight  (reduce + (map :similarity relevant-users))]
    
    (if (zero? total-weight)
      nil  ; Cannot predict
      (/ weighted-sum total-weight))))

;; Get top recommendations for user
(defn recommend-items [ratings user-id n]
  (let [user-ratings  (get ratings user-id #{})
        all-items     (set (mapcat keys (vals ratings)))
        unrated-items (remove #(contains? user-ratings %) all-items)]
    
    (->> unrated-items
         (map (fn [item]
                {:item   item
                 :score  (or (predict-rating ratings user-id item 5) 0.0)}))
         (sort-by :score >)
         (take n)
         (map :item))))

;; Example
(def user-ratings
  {"alice" {"item1" 5 "item2" 3 "item3" 4}
   "bob"   {"item1" 4 "item2" 5 "item4" 3}
   "carol" {"item1" 3 "item3" 5 "item4" 4 "item5" 2}})

(recommend-items user-ratings "alice" 3)
```

---

## ขั้นตอนที่ 1864: Text Classification

```clojure
;; Naive Bayes text classifier
(defn tokenize [text]
  (-> text
      clojure.string/lower-case
      (clojure.string/replace #"[^a-z\s]" "")
      (clojure.string/split #"\s+")
      (->> (remove #(contains? #{"the" "a" "an" "is" "it" "in" "on" "at"} %))
           (remove clojure.string/blank?))))

(defn train-naive-bayes [training-data]
  (let [classes        (group-by :class training-data)
        class-counts   (map-vals count classes)
        total-docs     (count training-data)
        
        class-probs    (map-vals #(/ % (double total-docs)) class-counts)
        
        word-counts    (map-vals
                          (fn [docs]
                            (frequencies (mapcat #(tokenize (:text %)) docs)))
                          classes)
        
        vocab-size     (count (set (mapcat #(keys (second %)) word-counts)))]
    
    {:class-probs class-probs
     :word-counts word-counts
     :vocab-size  vocab-size}))

(defn classify [model text]
  (let [words      (tokenize text)
        class-scores
        (into {}
          (for [[class prob] (:class-probs model)]
            [class
             (+ (Math/log prob)
                (reduce + (map
                             (fn [word]
                               (let [word-count (get-in model [:word-counts class word] 0)
                                     total-words (reduce + (vals (get-in model [:word-counts class] {})))]
                                 ;; Laplace smoothing
                                 (Math/log (/ (inc word-count)
                                               (+ total-words (:vocab-size model))))))
                             words)))]))]
    
    (key (apply max-key val class-scores))))

;; Training data for sentiment analysis
(def training-data
  [{:text "great product love it" :class :positive}
   {:text "excellent quality fast shipping" :class :positive}
   {:text "terrible product broken" :class :negative}
   {:text "waste of money disappointed" :class :negative}
   {:text "okay average nothing special" :class :neutral}])

(def model (train-naive-bayes training-data))

(classify model "amazing quality great value")  ; => :positive
(classify model "broken and useless")           ; => :negative
```

---

## ขั้นตอนที่ 1865: Feature Engineering

```clojure
;; Feature engineering pipeline for ML
(defn normalize-min-max [features]
  (let [mins (mapv #(apply min %) (apply mapv vector features))
        maxs (mapv #(apply max %) (apply mapv vector features))]
    (map (fn [row]
           (mapv (fn [v lo hi]
                    (if (= lo hi) 0.0
                        (/ (- v lo) (- hi lo))))
                  row mins maxs))
         features)))

(defn one-hot-encode [category all-categories]
  (mapv #(if (= % category) 1 0) all-categories))

(defn polynomial-features [features degree]
  (mapv (fn [row]
           (concat row
             (for [i (range (count row))
                   j (range i (count row))
                   :when (<= 2 degree)]
               (* (nth row i) (nth row j)))))
         features))

;; Feature extraction from orders
(defn extract-order-features [order history]
  (let [order-count    (count history)
        avg-value      (if (empty? history) 0
                         (/ (reduce + (map :total history)) order-count))
        days-since-last (if (empty? history) 365
                          (days-between (:created-at (last history))
                                         (java.time.LocalDate/now)))]
    [order-count
     avg-value
     days-since-last
     (:total order)
     (count (:items order))]))
```

---

## Project: Product Recommendation Engine

```clojure
(ns myapp.recommendations
  (:require [next.jdbc :as jdbc]))

;; Load ratings from DB
(defn load-ratings [db]
  (->> (jdbc/execute! db
         ["SELECT user_id, product_id, rating FROM product_ratings"])
       (group-by :product_ratings/user_id)
       (map-vals (fn [rows]
                    (into {}
                      (map (fn [r]
                               [(str (:product_ratings/product_id r))
                                (:product_ratings/rating r)])
                           rows))))))

;; Build and cache recommendation model
(defonce recommendation-model
  (atom nil))

(defn refresh-model! [db]
  (let [ratings (load-ratings db)
        model   {:ratings    ratings
                 :updated-at (java.time.Instant/now)}]
    (reset! recommendation-model model)
    (println "Recommendation model updated with" (count ratings) "users")))

;; Get recommendations
(defn get-recommendations [user-id n]
  (if-let [model @recommendation-model]
    (recommend-items (:ratings model) user-id n)
    []))

;; Schedule model refresh
(defn start-model-refresh! [db interval-hours]
  (future
    (loop []
      (refresh-model! db)
      (Thread/sleep (* interval-hours 3600000))
      (recur))))
```

---

*Part 63 จาก 100+ | ขั้นตอน 1861-1890 จาก 1000+*
