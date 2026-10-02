# Part 35: Java Interoperability
## ขั้นตอนที่ 1021-1050: Java Interop, Libraries, Reflection, AOT Compilation

---

## บทนำ

Clojure รันบน JVM → สามารถใช้ Java libraries ทั้งหมด:
- เรียก Java methods ตรงๆ
- Implement Java interfaces
- Extend Java classes
- ใช้ Java generics
- AOT compilation สำหรับ startup performance

---

## ขั้นตอนที่ 1021: Basic Java Interop

```clojure
;; Method call: .method
(.toUpperCase "hello")     ; => "HELLO"
(.length "hello")           ; => 5
(.substring "hello" 1 3)   ; => "el"

;; Static method: Class/method
(Math/sqrt 16.0)           ; => 4.0
(Math/pow 2 10)            ; => 1024.0
(System/currentTimeMillis) ; => timestamp
(Integer/parseInt "42")    ; => 42

;; Field access: .-field
(.-pi Math)                ; doesn't work - it's Math/PI
Math/PI                    ; => 3.14159...

;; Constructor: (ClassName. args)
(java.util.Date.)
(java.util.ArrayList. 10)
(StringBuilder.)

;; Chaining
(-> (StringBuilder.)
    (.append "Hello")
    (.append " ")
    (.append "World")
    (.toString))
;; => "Hello World"

;; Doto: call multiple methods on same object
(doto (java.util.HashMap.)
  (.put "key1" "value1")
  (.put "key2" "value2"))
```

---

## ขั้นตอนที่ 1022: Collections Interop

```clojure
;; Java → Clojure
(java.util.Arrays/asList (object-array [1 2 3]))
;; => java.util.Arrays$ArrayList

(into [] (java.util.Arrays/asList (object-array [1 2 3])))
;; => [1 2 3]

;; Clojure → Java
(into-array [1 2 3])          ; Object array
(int-array [1 2 3])           ; int[]
(double-array [1.0 2.0 3.0]) ; double[]

;; Convert Clojure map to Java Map
(java.util.HashMap. {:a 1 :b 2})

;; Convert to java.util.List
(java.util.ArrayList. [1 2 3])

;; Iterate Java Iterable
(doseq [item (java.util.Arrays/asList (object-array ["a" "b" "c"]))]
  (println item))

;; Java Stream API (Java 8+)
(-> (java.util.Arrays/stream (int-array [1 2 3 4 5]))
    (.filter (reify java.util.function.IntPredicate
               (test [_ n] (even? n))))
    (.map    (reify java.util.function.IntUnaryOperator
               (applyAsInt [_ n] (* n n))))
    (.sum))
;; => 20 (4 + 16)

;; Modern approach: seq works on most Java collections
(seq (java.util.Arrays/asList (object-array [1 2 3])))
;; => (1 2 3)
```

---

## ขั้นตอนที่ 1023: Implementing Java Interfaces

```clojure
;; reify: create anonymous implementation of interface(s)

;; Comparable
(defn make-comparable-point [x y]
  (reify
    java.lang.Comparable
    (compareTo [this other]
      (compare (* x x (* y y))
               (* (.getX other) (.getX other) (* (.getY other) (.getY other)))))
    
    Object
    (toString [_] (str "(" x "," y ")"))
    (equals [this other]
      (and (= x (.getX other))
           (= y (.getY other))))
    (hashCode [_] (hash [x y]))))

;; Runnable
(defn make-task [f]
  (reify Runnable (run [_] (f))))

(let [task (make-task #(println "Running in thread!"))]
  (.start (Thread. task)))

;; Java functional interfaces
(reify java.util.function.Function
  (apply [_ x] (* x 2)))

;; Comparator
(sort (reify java.util.Comparator
        (compare [_ a b] (compare (count a) (count b))))
      ["banana" "fig" "apple" "date"])
;; => ["fig" "date" "apple" "banana"]
```

---

## ขั้นตอนที่ 1024: Extending Java Classes ด้วย proxy

```clojure
;; proxy: extend Java class
;; ใช้เมื่อต้องการ override methods หรือ inherit state

(defn make-logged-thread [f label]
  (proxy [Thread] []
    (run []
      (println label "started")
      (f)
      (println label "finished"))
    
    (toString []
      (str "LoggedThread[" label "]"))))

;; HttpServlet extension
(defn make-servlet [handler]
  (proxy [javax.servlet.http.HttpServlet] []
    (doGet [request response]
      (handler request response))
    (doPost [request response]
      (handler request response))))

;; Override hashCode and equals
(defn make-entity [id name]
  (proxy [Object] []
    (hashCode [] (hash id))
    (equals  [other]
      (and (instance? (class this) other)
           (= id (.id other))))
    (toString [] (str "Entity[" id ":" name "]"))))
```

---

## ขั้นตอนที่ 1025: Type Hints สำหรับ Performance

```clojure
;; Type hints ขจัด reflection (ช้า) → ทำให้เร็วขึ้นมาก

;; Enable reflection warnings
(set! *warn-on-reflection* true)

;; Without hints (reflection = slow)
(defn no-hints [s n]
  (.substring s 0 n))

;; With hints (no reflection = fast)
(defn with-hints [^String s ^long n]
  (.substring s 0 (int n)))

;; Primitive type hints
(defn sum-array [^doubles arr]
  (areduce arr i total 0.0
    (+ total (aget arr i))))

;; Return type hint
(defn ^String get-name [^java.util.Map m]
  (.get m "name"))

;; Benchmarking
(require '[criterium.core :as crit])
(crit/bench (no-hints "hello world" 5))  ; ~50ns with reflection
(crit/bench (with-hints "hello world" 5)) ; ~5ns without reflection
```

---

## ขั้นตอนที่ 1026: Java Libraries Integration

```clojure
;; ===== Apache Commons =====
(import '[org.apache.commons.lang3 StringUtils])
(StringUtils/capitalize "hello world")       ; => "Hello world"
(StringUtils/abbreviate "long string" 10)    ; => "Long st..."
(StringUtils/isBlank "  ")                  ; => true

;; ===== Guava =====
(import '[com.google.common.collect ImmutableMap ImmutableList])
(def m (.build (-> (ImmutableMap/builder)
                    (.put "a" 1)
                    (.put "b" 2))))

;; ===== Joda Time (legacy) / java.time (modern) =====
(import '[java.time LocalDate LocalDateTime ZonedDateTime ZoneId Duration])

(def now (LocalDateTime/now))
(def bangkok (ZoneId/of "Asia/Bangkok"))
(def now-bangkok (ZonedDateTime/now bangkok))

(.format now-bangkok
  (java.time.format.DateTimeFormatter/ofPattern "yyyy-MM-dd HH:mm:ss z"))

;; Date arithmetic
(-> (LocalDate/now)
    (.plusDays 7)
    (.plusMonths 1))

;; Parse date
(LocalDate/parse "2024-01-15")

;; Duration
(def one-hour (Duration/ofHours 1))
(.toMinutes one-hour)  ; => 60
```

---

## ขั้นตอนที่ 1027: AOT Compilation

```clojure
;; Ahead-of-Time compilation: compile Clojure → .class files
;; Benefits: faster startup, smaller runtime

;; deps.edn build.clj
(ns build
  (:require [clojure.tools.build.api :as b]))

(defn compile-aot []
  (b/compile-clj
    {:basis     (b/create-basis {:project "deps.edn"})
     :src-dirs  ["src"]
     :class-dir "target/classes"
     :ns-compile '[myapp.core  ; Entry point ns
                   myapp.routes
                   myapp.db]}))  ; Namespaces to AOT

;; Namespace to AOT must declare :gen-class or use records/types
(ns myapp.core
  (:gen-class))

(defn -main [& args]
  (println "Starting application...")
  (start-system!))

;; Check what's being reflected (AOT won't fix if already fast)
(set! *warn-on-reflection* true)
(compile 'myapp.core)
;; Warnings show where reflection happens
```

---

## Project: PDF Generation ด้วย Apache PDFBox

```clojure
(ns myapp.pdf
  (:import [org.apache.pdfbox.pdmodel PDDocument PDPage PDPageContentStream]
           [org.apache.pdfbox.pdmodel.common PDRectangle]
           [org.apache.pdfbox.pdmodel.font PDType1Font Standard14Fonts$FontName]))

(defn create-invoice-pdf [invoice output-path]
  (with-open [doc (PDDocument.)]
    (let [page    (PDPage. PDRectangle/A4)
          _       (.addPage doc page)
          content (PDPageContentStream. doc page)]
      
      ;; Title
      (.beginText content)
      (.setFont content (PDType1Font. Standard14Fonts$FontName/HELVETICA_BOLD) 24)
      (.newLineAtOffset content 50 750)
      (.showText content "ใบแจ้งหนี้")
      (.endText content)
      
      ;; Invoice details
      (.beginText content)
      (.setFont content (PDType1Font. Standard14Fonts$FontName/HELVETICA) 12)
      (.newLineAtOffset content 50 700)
      (.showText content (str "เลขที่: " (:id invoice)))
      (.newLine content)
      (.showText content (str "วันที่: " (:date invoice)))
      (.newLine content)
      (.showText content (str "ลูกค้า: " (get-in invoice [:customer :name])))
      (.endText content)
      
      ;; Items table
      (let [y (atom 620)]
        (doseq [item (:items invoice)]
          (.beginText content)
          (.newLineAtOffset content 50 @y)
          (.showText content (str (:name item) " x " (:qty item)
                                   "  = " (:subtotal item) " บาท"))
          (.endText content)
          (swap! y - 20)))
      
      ;; Total
      (.beginText content)
      (.setFont content (PDType1Font. Standard14Fonts$FontName/HELVETICA_BOLD) 14)
      (.newLineAtOffset content 50 (- @y 20))
      (.showText content (str "รวมทั้งสิ้น: " (:total invoice) " บาท"))
      (.endText content)
      
      (.close content)
      (.save doc output-path))))
```

---

*Part 35 จาก 100+ | ขั้นตอน 1021-1050 จาก 1000+*
