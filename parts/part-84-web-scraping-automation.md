# Part 84: Web Scraping และ Browser Automation
## ขั้นตอนที่ 2491-2520: HTTP Client, HTML Parsing, Playwright, Data Extraction

---

## บทนำ

Web automation ด้วย Clojure:
- **hato** - HTTP client (Java 11 HttpClient)
- **Hickory/Enlive** - HTML parsing and selection
- **Etaoin** - WebDriver/Selenium wrapper
- **Data extraction** - structured data from HTML
- **Rate limiting** - respect robots.txt

---

## ขั้นตอนที่ 2491: HTTP Client ด้วย hato

```clojure
(ns myapp.scraping.http
  (:require [hato.client :as http]
            [hato.middleware :as hm]))

;; Build client with configuration
(def http-client
  (http/build-http-client
    {:connect-timeout 10000
     :follow-redirects :always
     :cookie-policy    :accept-all}))

;; GET request
(defn fetch-page [url]
  (http/get url
    {:http-client http-client
     :headers     {"User-Agent"      "Mozilla/5.0 (compatible; MyBot/1.0)"
                    "Accept"          "text/html,application/xhtml+xml"
                    "Accept-Language" "en-US,en;q=0.9"}
     :timeout     30000}))

;; POST with form data
(defn submit-form [url form-data]
  (http/post url
    {:form-params form-data
     :as          :json}))

;; JSON API
(defn fetch-json [url]
  (http/get url
    {:as            :json
     :http-client   http-client
     :oauth-token   (System/getenv "API_TOKEN")}))

;; Retry on failure
(defn fetch-with-retry [url max-retries]
  (loop [retries 0]
    (let [result (try
                   {:ok (fetch-page url)}
                   (catch Exception e
                     {:error e}))]
      (cond
        (:ok result) (:ok result)
        (>= retries max-retries) (throw (:error result))
        :else (do
                (Thread/sleep (* 1000 (inc retries)))
                (recur (inc retries)))))))

;; Parallel fetching with rate limiting
(defn fetch-many [urls max-concurrent delay-ms]
  (let [semaphore (java.util.concurrent.Semaphore. max-concurrent)]
    (pmap (fn [url]
            (.acquire semaphore)
            (Thread/sleep delay-ms)
            (try
              (fetch-page url)
              (finally
                (.release semaphore))))
          urls)))
```

---

## ขั้นตอนที่ 2492: HTML Parsing ด้วย Hickory

```clojure
(ns myapp.scraping.parser
  (:require [hickory.core :as h]
            [hickory.select :as s]))

;; Parse HTML
(defn parse-html [html-string]
  (-> html-string
      h/parse
      h/as-hickory))

;; CSS selectors
(defn select [tree selector]
  (s/select selector tree))

;; Extract product data from e-commerce page
(defn extract-products [html]
  (let [tree  (parse-html html)
        items (select tree (s/class "product-card"))]
    (map (fn [item]
           {:name    (-> item
                         (select (s/class "product-name"))
                         first
                         :content
                         first)
            :price   (-> item
                         (select (s/class "product-price"))
                         first
                         :content
                         first
                         (clojure.string/replace #"[^0-9.]" "")
                         Double/parseDouble)
            :url     (-> item
                         (select (s/tag :a))
                         first
                         (get-in [:attrs :href]))
            :img-url (-> item
                         (select (s/tag :img))
                         first
                         (get-in [:attrs :src]))})
         items)))

;; Text extraction
(defn extract-text [node]
  (cond
    (string? node) node
    (map? node)    (apply str (map extract-text (:content node [])))
    :else          ""))

;; Find all links
(defn extract-links [tree base-url]
  (let [anchors (select tree (s/tag :a))]
    (->> anchors
         (map #(get-in % [:attrs :href]))
         (filter identity)
         (map (fn [href]
                (cond
                  (clojure.string/starts-with? href "http") href
                  (clojure.string/starts-with? href "/")    (str base-url href)
                  :else (str base-url "/" href))))
         distinct)))
```

---

## ขั้นตอนที่ 2493: Full Web Scraper

```clojure
(ns myapp.scraping.crawler)

;; Site crawler with politeness
(defrecord Crawler [http-client db visited-urls queue config])

(defn make-crawler [config db]
  (map->Crawler
    {:http-client (http/build-http-client {})
     :db          db
     :visited-urls (atom #{})
     :queue       (java.util.concurrent.ConcurrentLinkedQueue.)
     :config      (merge {:max-pages      1000
                           :delay-ms       1000
                           :max-concurrent 5
                           :allowed-domains #{}} config)}))

(defn should-crawl? [crawler url]
  (let [{:keys [visited-urls config]} crawler
        domain (-> (java.net.URI. url) .getHost)]
    (and
      (not (contains? @visited-urls url))
      (or (empty? (:allowed-domains config))
          (contains? (:allowed-domains config) domain)))))

(defn crawl-page! [crawler url]
  (when (should-crawl? crawler url)
    (swap! (:visited-urls crawler) conj url)
    
    (try
      (let [response (fetch-page url)
            html     (:body response)
            tree     (parse-html html)
            text     (extract-text (select tree (s/tag :body)))
            links    (extract-links tree url)]
        
        ;; Save page data
        (save-page! (:db crawler) {:url      url
                                    :html     html
                                    :text     text
                                    :crawled-at (java.time.Instant/now)})
        
        ;; Queue new links
        (doseq [link links]
          (when (< (count @(:visited-urls crawler))
                    (get-in crawler [:config :max-pages]))
            (.offer (:queue crawler) link)))
        
        {:url url :links (count links) :success? true})
      
      (catch Exception e
        {:url url :error (.getMessage e) :success? false}))))

;; Run crawler
(defn start-crawl! [crawler seed-url]
  (.offer (:queue crawler) seed-url)
  (let [executor (java.util.concurrent.Executors/newFixedThreadPool
                   (get-in crawler [:config :max-concurrent]))]
    (loop []
      (when-let [url (.poll (:queue crawler))]
        (Thread/sleep (get-in crawler [:config :delay-ms]))
        (.submit executor #(crawl-page! crawler url))
        (recur)))))
```

---

## ขั้นตอนที่ 2494: Structured Data Extraction

```clojure
;; Extract structured data from HTML

;; JSON-LD extraction
(defn extract-json-ld [html]
  (let [tree   (parse-html html)
        scripts (select tree
                   (s/and (s/tag :script)
                          (s/attr :type #(= % "application/ld+json"))))]
    (->> scripts
         (map (comp first :content))
         (filter identity)
         (map #(json/parse-string % true)))))

;; Product schema.org extraction
(defn extract-product-schema [html]
  (->> (extract-json-ld html)
       (filter #(= "Product" (:@type %)))
       first
       (fn [schema]
         {:name        (:name schema)
          :description (:description schema)
          :price       (get-in schema [:offers :price])
          :currency    (get-in schema [:offers :priceCurrency])
          :sku         (:sku schema)
          :image       (:image schema)})))

;; Table extraction
(defn extract-table [html table-selector]
  (let [tree  (parse-html html)
        table (first (select tree table-selector))
        rows  (select table (s/tag :tr))]
    (let [headers (map extract-text
                       (select (first rows) (s/tag :th)))
          data-rows (rest rows)]
      (map (fn [row]
              (zipmap (map keyword headers)
                       (map extract-text (select row (s/tag :td)))))
           data-rows))))
```

---

*Part 84 จาก 100+ | ขั้นตอน 2491-2520 จาก 1000+*
