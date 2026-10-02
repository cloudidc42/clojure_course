# Part 60: Search Engine และ Elasticsearch
## ขั้นตอนที่ 1771-1800: Full-Text Search, Indexing, Aggregations, Auto-complete

---

## บทนำ

Search capabilities สำหรับ production:
- **Elasticsearch** - distributed search engine
- **Full-text search** - Thai/English text analysis
- **Aggregations** - faceted search, analytics
- **Auto-complete** - suggest as you type
- **PostgreSQL FTS** - built-in full-text search

---

## ขั้นตอนที่ 1771: Elasticsearch Client

```clojure
(ns myapp.search
  (:require [clj-http.client :as http]
            [cheshire.core :as json]))

;; Low-level ES client
(defn es-request!
  [method path body es-url]
  (let [response (http/request
                    {:method       method
                     :url          (str es-url path)
                     :headers      {"Content-Type" "application/json"}
                     :body         (when body (json/generate-string body))
                     :as           :json
                     :coerce       :always})]
    (:body response)))

(defn create-index! [es-url index-name settings]
  (es-request! :put (str "/" index-name) settings es-url))

(defn index-document! [es-url index-name doc-id doc]
  (es-request! :put (str "/" index-name "/_doc/" doc-id) doc es-url))

(defn bulk-index! [es-url index-name docs]
  (let [body (mapcat (fn [doc]
                        [{"index" {"_index" index-name
                                    "_id"    (str (:id doc))}}
                         doc])
                      docs)
        body-str (str (clojure.string/join "\n"
                                            (map json/generate-string body))
                       "\n")]
    (http/post (str es-url "/_bulk")
      {:headers {"Content-Type" "application/x-ndjson"}
       :body    body-str
       :as      :json})))

(defn search! [es-url index-name query]
  (es-request! :post (str "/" index-name "/_search") query es-url))

(defn delete-document! [es-url index-name doc-id]
  (es-request! :delete (str "/" index-name "/_doc/" doc-id) nil es-url))
```

---

## ขั้นตอนที่ 1772: Index Mapping

```clojure
;; Product index mapping
(def product-index-settings
  {:settings
   {:number_of_shards   2
    :number_of_replicas 1
    :analysis
    {:analyzer
     {:product-analyzer
      {:type      "custom"
       :tokenizer "standard"
       :filter    ["lowercase" "stop" "english_stemmer"]}
      :ngram-analyzer
      {:type      "custom"
       :tokenizer "ngram_tokenizer"
       :filter    ["lowercase"]}}
     :tokenizer
     {:ngram_tokenizer
      {:type       "ngram"
       :min_gram   2
       :max_gram   3
       :token_chars ["letter" "digit"]}}
     :filter
     {:english_stemmer
      {:type     "stemmer"
       :language "english"}}}}
   :mappings
   {:properties
    {:id          {:type "keyword"}
     :name        {:type     "text"
                    :analyzer "product-analyzer"
                    :fields   {:keyword  {:type "keyword"}
                                :suggest  {:type "completion"}
                                :ngram    {:type "text" :analyzer "ngram-analyzer"}}}
     :description {:type "text" :analyzer "product-analyzer"}
     :category    {:type "keyword"}
     :brand       {:type "keyword"}
     :price       {:type "float"}
     :stock       {:type "integer"}
     :tags        {:type "keyword"}
     :created_at  {:type "date"}
     :active      {:type "boolean"}}}})

(defn setup-product-index! [es-url]
  (create-index! es-url "products" product-index-settings))
```

---

## ขั้นตอนที่ 1773: Full-Text Search Queries

```clojure
(ns myapp.search.queries)

;; Multi-match search with boosting
(defn search-products [es-url query-str & {:keys [page size category min-price max-price sort-by]
                                            :or {page 1 size 20}}]
  (let [must-clauses
        [(when query-str
           {:multi_match
            {:query  query-str
             :fields ["name^3"          ; boost name 3x
                      "name.ngram^2"    ; ngram for partial match
                      "description"
                      "tags^2"]
             :type   "best_fields"
             :fuzziness "AUTO"}})]
        
        filter-clauses
        (cond-> []
          category  (conj {:term {:category category}})
          (or min-price max-price)
          (conj {:range {:price (cond-> {}
                                   min-price (assoc :gte min-price)
                                   max-price (assoc :lte max-price))}})
          true (conj {:term {:active true}}))
        
        query {:query
               {:bool {:must   (remove nil? must-clauses)
                        :filter filter-clauses}}
                
                :sort
                (case sort-by
                  "price-asc"   [{:price {:order "asc"}}]
                  "price-desc"  [{:price {:order "desc"}}]
                  "newest"      [{:created_at {:order "desc"}}]
                  [{:_score {:order "desc"}}])
                
                :from (* (dec page) size)
                :size size
                
                :highlight
                {:fields {:name        {}
                           :description {:number_of_fragments 2
                                          :fragment_size 100}}}
                
                :_source true}]
    
    (let [response (search! es-url "products" query)
          hits     (get-in response [:hits :hits])]
      {:total   (get-in response [:hits :total :value])
       :items   (map (fn [hit]
                        (merge (:_source hit)
                                {:_score     (:_score hit)
                                 :highlights (get-in hit [:highlight])}))
                      hits)
       :page    page
       :pages   (Math/ceil (/ (get-in response [:hits :total :value]) size))})))
```

---

## ขั้นตอนที่ 1774: Aggregations สำหรับ Faceted Search

```clojure
;; Faceted search with aggregations
(defn search-with-facets [es-url query-str filters]
  (let [query {:query
               {:bool
                {:must   [{:multi_match {:query query-str :fields ["name" "description"]}}]
                 :filter (map (fn [[field value]]
                                 {:term {field value}})
                               filters)}}
                
                :aggs
                {:categories
                 {:terms {:field "category" :size 20}
                  :aggs  {:avg_price {:avg {:field "price"}}}}
                 
                 :price_ranges
                 {:range {:field "price"
                           :ranges [{:key "under-10"  :to  10.0}
                                     {:key "10-50"     :from 10.0 :to 50.0}
                                     {:key "50-100"    :from 50.0 :to 100.0}
                                     {:key "over-100"  :from 100.0}]}}
                 
                 :brands
                 {:terms {:field "brand" :size 10}}
                 
                 :top_tags
                 {:terms {:field "tags" :size 20}}}
                
                :size 20}
        
        response (search! es-url "products" query)]
    
    {:results (map :_source (get-in response [:hits :hits]))
     :total   (get-in response [:hits :total :value])
     :facets  {:categories  (get-in response [:aggregations :categories :buckets])
                :price-ranges (get-in response [:aggregations :price_ranges :buckets])
                :brands      (get-in response [:aggregations :brands :buckets])
                :tags        (get-in response [:aggregations :top_tags :buckets])}}))
```

---

## ขั้นตอนที่ 1775: Auto-complete

```clojure
;; Completion suggester for autocomplete
(defn suggest-products [es-url prefix]
  (let [query {:suggest
               {:product-suggest
                {:prefix  prefix
                 :completion
                 {:field "name.suggest"
                  :size  10
                  :fuzzy {:fuzziness "AUTO"}}}}}
        
        response (search! es-url "products" query)]
    
    (->> (get-in response [:suggest :product-suggest 0 :options])
         (map (fn [opt]
                {:text  (:text opt)
                 :score (:_score opt)
                 :data  (:_source opt)})))))

;; Index documents with suggest field
(defn product->index-doc [product]
  {:id          (:id product)
   :name        (:name product)
   :description (:description product)
   :category    (:category product)
   :brand       (:brand product)
   :price       (:price product)
   :stock       (:stock product)
   :tags        (:tags product)
   :active      (:active? product true)
   :created_at  (str (:created-at product))
   ;; Suggest field
   :name.suggest {:input   [(:name product)
                              ;; Add brand as alternative input
                              (str (:brand product) " " (:name product))]
                   :weight  (if (:featured? product) 10 1)}})

;; Sync from DB to ES
(defn sync-products! [db es-url]
  (let [products (jdbc/execute! db
                   ["SELECT * FROM products WHERE active = TRUE"])]
    (bulk-index! es-url "products"
      (map product->index-doc products))))
```

---

## Project: E-Commerce Search Service

```clojure
(ns search.service
  (:require [ring.core.middleware :as middleware]))

;; Search API endpoints
(def search-routes
  [["/search"
    {:get {:handler
           (fn [request]
             (let [q       (get-in request [:query-params "q"])
                   page    (Integer/parseInt (get-in request [:query-params "page"] "1"))
                   category (get-in request [:query-params "category"])]
               {:status 200
                :body   (search-products es-url q
                          :page page
                          :category category)}))}}]
   
   ["/search/suggest"
    {:get {:handler
           (fn [request]
             (let [prefix (get-in request [:query-params "q"] "")]
               {:status 200
                :body   {:suggestions (suggest-products es-url prefix)}}))}}]
   
   ["/search/facets"
    {:get {:handler
           (fn [request]
             (let [q       (get-in request [:query-params "q"] "")
                   filters {}]
               {:status 200
                :body   (search-with-facets es-url q filters)}))}}]])

;; Keep ES in sync with DB changes
(defn setup-sync! [db es-url]
  ;; Listen to DB changes (using LISTEN/NOTIFY or CDC)
  (subscribe! :product-created
    (fn [event]
      (index-document! es-url "products"
        (str (:id (:data event)))
        (product->index-doc (:data event)))))
  
  (subscribe! :product-updated
    (fn [event]
      (index-document! es-url "products"
        (str (:id (:data event)))
        (product->index-doc (:data event)))))
  
  (subscribe! :product-deleted
    (fn [event]
      (delete-document! es-url "products" (str (:id (:data event)))))))
```

---

*Part 60 จาก 100+ | ขั้นตอน 1771-1800 จาก 1000+*
