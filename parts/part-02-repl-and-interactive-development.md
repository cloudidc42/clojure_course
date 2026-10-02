# Part 2: REPL และ Interactive Development
## ขั้นตอนที่ 31-60: เชี่ยวชาญการพัฒนาแบบ Interactive

---

## บทนำ

REPL (Read-Eval-Print Loop) ไม่ใช่แค่ "Console" ธรรมดา - มันคือ **วิธีคิดในการพัฒนาซอฟต์แวร์** ที่แตกต่างจากทุกภาษาที่คุณเคยใช้มา

ใน Clojure คุณไม่ได้ "เขียนโค้ดแล้วรัน" แต่คุณ **"สนทนากับโปรแกรม"** ขณะที่มันทำงานอยู่

---

## ขั้นตอนที่ 31: REPL คืออะไรจริงๆ?

```
Traditional Development:
  Edit → Compile → Run → Debug → Edit → Compile → ...
  (ใช้เวลาหลายวินาทีถึงหลายนาที per cycle)

REPL-Driven Development:
  Think → Evaluate → Observe → Refine
  (ใช้เวลาน้อยกว่า 1 วินาที per cycle!)
```

### ตัวอย่างเปรียบเทียบ

```java
// Java: ต้อง compile แล้วรันทั้งโปรแกรม
public class Main {
    public static void main(String[] args) {
        // ทดสอบ logic ทุกครั้งต้อง recompile + rerun
        System.out.println(calculateSomething(42));
    }
}
```

```clojure
;; Clojure REPL: ทดสอบทีละ expression ทันที
user=> (calculate-something 42)
;; เห็นผลทันที! ไม่ต้อง compile หรือ rerun

user=> (defn calculate-something [x] (* x x))
;; define แล้วทดสอบทันที

user=> (calculate-something 42)
1764

user=> (map calculate-something [1 2 3 4 5])
(1 4 9 16 25)
```

---

## ขั้นตอนที่ 32: เปิด REPL ด้วยวิธีต่างๆ

```bash
# วิธีที่ 1: Clojure CLI
clojure
# หรือ
clj

# วิธีที่ 2: Leiningen
lein repl

# วิธีที่ 3: ใน project (แนะนำ)
cd my-project
lein repl

# วิธีที่ 4: nREPL Server (สำหรับ editor connection)
lein repl :headless :port 7888
# แล้วเชื่อม editor เข้า port 7888
```

### Prompt ที่เห็น

```
user=>     ; namespace ปัจจุบัน (user)
dev=>      ; ถ้าอยู่ใน dev namespace
my.app=>   ; namespace ของ project
```

---

## ขั้นตอนที่ 33: Navigation ใน REPL

```clojure
;; ดู namespace ปัจจุบัน
*ns*
;; => #object[clojure.lang.Namespace 0x... "user"]

;; เปลี่ยน namespace
(in-ns 'my.namespace)

;; ดู vars ที่ defined ใน namespace
(ns-publics *ns*)
(ns-interns *ns*)

;; ดู vars ทั้งหมด (รวม private)
(dir user)

;; ดูเอกสารของ function
(doc println)
(doc map)
(doc +)

;; ดู source code ของ function
(source map)
(source filter)
```

---

## ขั้นตอนที่ 34: ฟีเจอร์พิเศษของ REPL

```clojure
;; *1, *2, *3 = ผลลัพธ์ล่าสุด 3 ครั้ง
(+ 1 2)   ; => 3
(+ 3 4)   ; => 7
(+ 5 6)   ; => 11
*1        ; => 11 (ล่าสุด)
*2        ; => 7  (ก่อนหน้า)
*3        ; => 3  (ก่อนหน้าอีก)

;; *e = exception ล่าสุด
(/ 1 0)   ; ArithmeticException
*e        ; ดู exception details

;; *print-length* = จำกัดการแสดง collection
(set! *print-length* 5)
(range 100)  ; => (0 1 2 3 4 ...)

;; *print-level* = จำกัด nesting depth
(set! *print-level* 2)
{:a {:b {:c {:d 1}}}}  ; => {:a {:b #}}
```

---

## ขั้นตอนที่ 35: Require และ Import ใน REPL

```clojure
;; โหลด namespace
(require '[clojure.string :as str])

;; ใช้งาน
(str/upper-case "hello")   ; => "HELLO"
(str/split "a,b,c" #",")  ; => ["a" "b" "c"]

;; โหลดหลาย namespaces
(require '[clojure.string :as str]
         '[clojure.set :as set]
         '[clojure.math :as math])

;; Reload หลังแก้โค้ด
(require '[my.namespace :as my] :reload)

;; Reload ทุก namespace ที่ dependency เปลี่ยน
(require '[my.namespace :as my] :reload-all)

;; Import Java classes
(import java.util.Date)
(import '[java.util Date Calendar])

(Date.)  ; สร้าง Date instance

;; use (ไม่แนะนำ แต่รู้ไว้)
;; (use 'clojure.string)  ; import ทุก public var
;; ปัญหา: ไม่รู้ว่ามาจากไหน ใช้ require :refer แทน

(require '[clojure.string :refer [upper-case lower-case]])
(upper-case "hello")  ; ใช้ได้โดยไม่ต้องใส่ str/
```

---

## ขั้นตอนที่ 36: ทดลองแบบ Incremental Development

นี่คือหัวใจของ REPL-driven development

```clojure
;; สมมติต้องการ process user data
;; เริ่มจากข้อมูลจำลอง

;; Step 1: สร้างข้อมูลทดสอบ
(def users
  [{:id 1 :name "สมชาย" :age 25 :salary 50000}
   {:id 2 :name "สมหญิง" :age 30 :salary 60000}
   {:id 3 :name "สมศักดิ์" :age 22 :salary 35000}
   {:id 4 :name "มาลี" :age 28 :salary 75000}])

;; Step 2: ดูข้อมูลที่มี
(first users)
;; => {:id 1, :name "สมชาย", :age 25, :salary 50000}

;; Step 3: ทดสอบ filter
(filter #(> (:salary %) 50000) users)
;; => ({:id 2, :name "สมหญิง", ...} {:id 4, :name "มาลี", ...})

;; Step 4: ทดสอบ map
(map :name users)
;; => ("สมชาย" "สมหญิง" "สมศักดิ์" "มาลี")

;; Step 5: รวมกัน
(->> users
     (filter #(> (:salary %) 50000))
     (map :name))
;; => ("สมหญิง" "มาลี")

;; Step 6: แปลงเป็น function
(defn high-earners [users min-salary]
  (->> users
       (filter #(> (:salary %) min-salary))
       (map :name)))

;; Step 7: ทดสอบทันที
(high-earners users 50000)
;; => ("สมหญิง" "มาลี")

(high-earners users 60000)
;; => ("มาลี")
```

---

## ขั้นตอนที่ 37: Namespace Management

```clojure
;; สร้างและใช้งาน namespace
(ns my.project.utils
  (:require [clojure.string :as str]))

;; ตอนนี้อยู่ใน my.project.utils namespace
(defn capitalize-words [s]
  (->> (str/split s #" ")
       (map str/capitalize)
       (str/join " ")))

;; กลับไป user namespace
(in-ns 'user)

;; โหลดและใช้
(require '[my.project.utils :as utils])
(utils/capitalize-words "hello world clojure")
;; => "Hello World Clojure"

;; ดู namespace structure
(ns-map 'my.project.utils)
(ns-publics 'my.project.utils)
```

---

## ขั้นตอนที่ 38: การใช้ doc และ find-doc

```clojure
;; ดู documentation
(doc map)
;; -------------------------
;; clojure.core/map
;; ([f] [f coll] [f c1 c2] [f c1 c2 c3] [f c1 c2 c3 & colls])
;;   Returns a lazy sequence consisting of the result of applying f to
;;   the set of first items of each coll, followed by applying f to the
;;   set of second items in each coll, until any one of the colls is
;;   exhausted.  Any remaining items in other colls are ignored. ...

;; ค้นหา function ที่ต้องการ
(find-doc "sort")
;; แสดงทุก function ที่มี "sort" ใน docstring

(find-doc "string")
;; แสดงทุก function ที่เกี่ยวกับ string

;; ดู source code
(source map)
(source reduce)
(source filter)

;; ดู type ของ value
(type 42)         ; => java.lang.Long
(type "hello")    ; => java.lang.String
(type [1 2 3])    ; => clojure.lang.PersistentVector
(type {:a 1})     ; => clojure.lang.PersistentArrayMap
(type nil)        ; => nil
(type println)    ; => clojure.core$println

;; class ของ Java
(class 42)        ; => java.lang.Long
```

---

## ขั้นตอนที่ 39: apropos และ dir

```clojure
;; หา function ที่ชื่อเหมือนกัน
(apropos "count")
;; => (clojure.core/bounded-count clojure.core/count ...)

(apropos "str")
;; => (clojure.core/str clojure.string/... ...)

;; ดู vars ใน namespace
(dir clojure.string)
;; blank?
;; capitalize
;; ends-with?
;; escape
;; includes?
;; index-of
;; join
;; last-index-of
;; lower-case
;; replace
;; replace-first
;; reverse
;; split
;; split-lines
;; starts-with?
;; trim
;; trim-newline
;; triml
;; trimr
;; upper-case

;; ดู vars ใน clojure.core
(dir clojure.core)
```

---

## ขั้นตอนที่ 40: pprint สำหรับ Pretty Printing

```clojure
(require '[clojure.pprint :as pp])

;; ข้อมูลซับซ้อน
(def complex-data
  {:users [{:name "สมชาย" :roles [:admin :user] :settings {:theme "dark" :lang "th"}}
           {:name "สมหญิง" :roles [:user] :settings {:theme "light" :lang "en"}}]
   :total 2
   :page 1})

;; println ธรรมดา (ยากอ่าน)
(println complex-data)

;; pprint (อ่านง่าย)
(pp/pprint complex-data)
;; {:users
;;  [{:name "สมชาย",
;;    :roles [:admin :user],
;;    :settings {:theme "dark", :lang "th"}}
;;   {:name "สมหญิง",
;;    :roles [:user],
;;    :settings {:theme "light", :lang "en"}}],
;;  :total 2,
;;  :page 1}

;; print-table
(pp/print-table [:name :age :salary]
  [{:name "สมชาย" :age 25 :salary 50000}
   {:name "สมหญิง" :age 30 :salary 60000}])
;; | :name  | :age | :salary |
;; |--------+------+---------|
;; | สมชาย  |   25 |   50000 |
;; | สมหญิง |   30 |   60000 |
```

---

## ขั้นตอนที่ 41: การ Inspect ข้อมูลใน REPL

```clojure
;; meta - ดู metadata ของ var
(meta #'map)
;; => {:arglists ([f] [f coll] ...), :doc "...", :added "1.0", ...}

(meta #'println)
;; => {:added "1.0", :ns #object[...], :name println, ...}

;; ดู metadata ของ collection
(def annotated-vec
  (with-meta [1 2 3] {:source "test" :timestamp 12345}))

(meta annotated-vec)
;; => {:source "test", :timestamp 12345}

;; inspect - ดู structure
(require '[clojure.inspector :as insp])
(insp/inspect complex-data)  ; เปิด Swing window
(insp/inspect-tree complex-data)  ; แบบ tree
```

---

## ขั้นตอนที่ 42: การ Load และ Reload Files

```clojure
;; load file
(load-file "src/my_app/core.clj")

;; require namespace
(require '[my-app.core :as core])

;; force reload
(require '[my-app.core :as core] :reload)

;; reload ทุกอย่าง
(require '[my-app.core :as core] :reload-all)

;; ใช้ use-fixtures ใน tests
;; ดูเพิ่มเติมใน Part 15: Testing
```

---

## ขั้นตอนที่ 43: Macros ใน REPL

```clojure
;; macroexpand - ดูว่า macro expand ออกมาเป็นอะไร
(macroexpand '(when true (println "hello")))
;; => (if true (do (println "hello")))

(macroexpand '(and a b c))
;; => (let* [and__... a] (if and__... (and b c) and__...))

(macroexpand-1 '(-> x f g h))
;; => (-> (f x) g h)  ; แค่ขั้นแรก

(macroexpand '(-> x f g h))
;; => (h (g (f x)))  ; expand ทั้งหมด

;; doto - multiple method calls on object
(macroexpand '(doto (StringBuilder.)
               (.append "Hello")
               (.append " World")))
;; แสดง expansion
```

---

## ขั้นตอนที่ 44: ทำงานกับ Java ใน REPL

```clojure
;; Import Java classes
(import java.util.Date)
(import java.time.LocalDateTime)
(import java.time.format.DateTimeFormatter)

;; สร้าง instance
(def now (LocalDateTime/now))

;; เรียก methods
(.getYear now)     ; => 2024
(.getMonthValue now) ; => 12
(.getDayOfMonth now) ; => 15

;; Static methods
(LocalDateTime/now)
(System/currentTimeMillis)
(Math/random)

;; Format date
(def formatter (DateTimeFormatter/ofPattern "dd/MM/yyyy HH:mm"))
(.format now formatter)
;; => "15/12/2024 14:30"

;; Java interop ครบ!
(import java.util.ArrayList)
(def list (ArrayList.))
(.add list "a")
(.add list "b")
(.size list)  ; => 2

;; แต่ใน Clojure เราชอบ Clojure collections มากกว่า
;; Java collections ใช้เมื่อจำเป็นเท่านั้น
```

---

## ขั้นตอนที่ 45: Scratch Namespace และ Dev Namespace

```clojure
;; สร้างไฟล์ dev/user.clj สำหรับ development utilities

;; dev/user.clj
(ns user
  (:require [clojure.tools.namespace.repl :as repl]
            [clojure.pprint :as pp]))

;; สั่ง reload ทุก namespace ที่เปลี่ยน
(defn refresh []
  (repl/refresh))

;; Stop/start system (สำหรับ component-based apps)
(defn reset []
  (repl/refresh-all))
```

```clojure
;; deps.edn - เพิ่ม dev profile
{:deps {org.clojure/clojure {:mvn/version "1.12.0"}}
 :paths ["src"]
 :aliases
 {:dev {:extra-paths ["dev"]
        :extra-deps
        {org.clojure/tools.namespace {:mvn/version "1.4.4"}
         nrepl/nrepl {:mvn/version "1.3.0"}}}}}
```

---

## ขั้นตอนที่ 46: ใช้ comment block สำหรับ Exploration

```clojure
;; src/my_app/core.clj

(ns my-app.core
  (:require [clojure.string :as str]))

(defn process-text [text]
  (->> text
       str/lower-case
       (re-seq #"\w+")
       frequencies
       (sort-by val >)))

;; Rich Comment Block - พื้นที่สำหรับทดลอง
(comment
  ;; ทดสอบกับข้อมูลจริง
  (process-text "Hello World Hello Clojure World World")
  ;; => (["world" 3] ["hello" 2] ["clojure" 1])
  
  ;; ทดสอบ edge cases
  (process-text "")
  ;; => ()
  
  (process-text "a a a b b c")
  ;; => (["a" 3] ["b" 2] ["c" 1])
  
  ;; ใน Calva กด Ctrl+Enter บน expression ใน comment เพื่อ eval
  )
```

---

## ขั้นตอนที่ 47: ดักจับ Error และ Stack Trace

```clojure
;; ทำให้เกิด error
(/ 1 0)
;; Execution error (ArithmeticException) at...
;; Divide by zero

;; อ่าน stack trace
(/ 1 0)
;; Error stack trace จะแสดงที่ REPL
;; อ่านจากบนลงล่าง หรือบางครั้งล่างขึ้นบน
;; มองหา บรรทัดที่เป็น code ของเรา (ไม่ใช่ clojure internal)

;; ใช้ pst สำหรับ pretty stack trace
(pst)  ; แสดง stack trace ของ error ล่าสุด

;; ดัก exception
(try
  (/ 1 0)
  (catch ArithmeticException e
    (println "Caught:" (.getMessage e))))
;; Caught: Divide by zero

;; ดู exception hierarchy
(.getSuperclass (class (Exception.)))
;; => java.lang.Throwable

;; ดู exception message
(try
  (throw (Exception. "test error"))
  (catch Exception e
    (println "Type:" (type e))
    (println "Message:" (.getMessage e))
    (println "Cause:" (.getCause e))))
```

---

## ขั้นตอนที่ 48: Performance Testing ใน REPL

```clojure
;; time - วัดเวลา
(time (Thread/sleep 100))
;; "Elapsed time: 100.xxx msecs"

(time (reduce + (range 1000000)))
;; "Elapsed time: 42.xxx msecs"
;; => 499999500000

;; criterium - สำหรับ accurate benchmarking
;; เพิ่มใน project.clj: [criterium "0.4.6"]

(require '[criterium.core :as bench])

(bench/bench (reduce + (range 1000)))
;; Evaluation count : 1560 in 60 samples of 26 calls.
;;              Execution time mean : 39.058027 µs
;;     Execution time std-deviation : 2.249889 µs
;; Execution time lower quantile : 36.720537 µs ( 2.5%)
;; Execution time upper quantile : 43.710905 µs (97.5%)

;; Quick bench (น้อยกว่า)
(bench/quick-bench (reduce + (range 1000)))
```

---

## ขั้นตอนที่ 49: Clojure REPL Utilities

```clojure
;; ดู Java system properties
(System/getProperty "java.version")
;; => "21.0.1"

(System/getProperty "user.home")
;; => "/home/username"

;; ดู environment variables
(System/getenv "PATH")
(System/getenv "HOME")

;; ดู JVM memory
(.totalMemory (Runtime/getRuntime))
(.freeMemory (Runtime/getRuntime))
(.maxMemory (Runtime/getRuntime))

;; Run garbage collection
(System/gc)

;; Exit
(System/exit 0)  ; ระวัง! ปิด JVM ด้วย

;; ดู Clojure version
(clojure-version)
;; => "1.12.0"

*clojure-version*
;; => {:major 1, :minor 12, :incremental 0, :qualifier nil}
```

---

## ขั้นตอนที่ 50: การตั้งค่า REPL ที่ดี

```clojure
;; ~/.clojure/deps.edn (Global config)
{:aliases
 {:repl {:extra-deps {nrepl/nrepl {:mvn/version "1.3.0"}
                      cider/cider-nrepl {:mvn/version "0.50.2"}
                      refactor-nrepl/refactor-nrepl {:mvn/version "3.10.0"}
                      com.bhauman/rebel-readline {:mvn/version "0.1.4"}}
         :main-opts ["-m" "nrepl.cmdline"
                     "--middleware" "[cider.nrepl/cider-middleware]"]}
  
  :portal {:extra-deps {djblue/portal {:mvn/version "0.58.0"}}}}}
```

```clojure
;; Portal - Data inspector for REPL
(require '[portal.api :as p])

(def portal (p/open))
(add-tap #'p/submit)

;; ส่งข้อมูลไปดูใน Portal browser
(tap> {:users [{:name "สมชาย" :age 25}
               {:name "สมหญิง" :age 30}]})
;; เปิด browser ดูข้อมูล!

;; Rebel Readline - REPL ที่ดีกว่า
;; clj -M:repl
;; มี syntax highlighting, autocomplete, history
```

---

## ขั้นตอนที่ 51: Workflow จริงใน REPL

```clojure
;; === Typical REPL Workflow ===

;; 1. เปิด REPL ใน project
;; lein repl

;; 2. โหลด namespace ที่กำลังทำงาน
(require '[my-app.core :as core] :reload-all)

;; 3. ดูข้อมูลตัวอย่าง
(def sample-data {:id 1 :name "test" :value 100})

;; 4. ทดลอง logic ทีละขั้น
(get sample-data :name)
;; => "test"

(assoc sample-data :processed true)
;; => {:id 1, :name "test", :value 100, :processed true}

;; 5. เขียนเป็น function
(defn process-item [item]
  (assoc item :processed true :timestamp (System/currentTimeMillis)))

;; 6. ทดสอบทันที
(process-item sample-data)
;; => {:id 1, :name "test", :value 100, :processed true, :timestamp 1234567890}

;; 7. Test หลาย cases
(map process-item [{:id 1} {:id 2} {:id 3}])

;; 8. Save ลง source file
;; (copy code ไปใส่ใน .clj file)

;; 9. Reload และ verify
(require '[my-app.core :as core] :reload)
(core/process-item sample-data)
```

---

## ขั้นตอนที่ 52: REPL-driven Testing

```clojure
;; ทดสอบโดยตรงใน REPL ก่อนเขียน formal tests

;; Function ที่ต้องทดสอบ
(defn parse-date [s]
  ;; parse "DD/MM/YYYY" to map
  (let [[d m y] (clojure.string/split s #"/")]
    {:day (Integer/parseInt d)
     :month (Integer/parseInt m)
     :year (Integer/parseInt y)}))

;; ทดสอบ happy path
(parse-date "15/12/2024")
;; => {:day 15, :month 12, :year 2024}

;; ทดสอบ edge cases
(parse-date "01/01/2024")
;; => {:day 1, :month 1, :year 2024}

;; ทดสอบ invalid input
(try
  (parse-date "invalid")
  (catch Exception e
    (println "Error:" (.getMessage e))))

;; เมื่อพอใจแล้ว เขียน formal tests
(require '[clojure.test :refer :all])

(deftest test-parse-date
  (is (= {:day 15 :month 12 :year 2024}
         (parse-date "15/12/2024")))
  (is (= {:day 1 :month 1 :year 2024}
         (parse-date "01/01/2024"))))

(run-tests)
```

---

## ขั้นตอนที่ 53: nREPL Protocol

nREPL (networked REPL) ทำให้ editor เชื่อมต่อกับ REPL ได้

```bash
# เริ่ม nREPL server
lein repl :headless :port 7888
# nREPL server started on port 7888 on host 127.0.0.1

# หรือ
clj -M -m nrepl.cmdline --port 7888
```

```clojure
;; เชื่อมต่อจาก Clojure code
(require '[nrepl.core :as nrepl])

(with-open [conn (nrepl/connect :port 7888)]
  (-> (nrepl/client conn 1000)
      (nrepl/message {:op "eval" :code "(+ 1 2)"})
      nrepl/response-values))
;; => [3]
```

---

## ขั้นตอนที่ 54: Calva REPL Integration

### VS Code + Calva Workflow

```
1. เปิด project ใน VS Code
2. Ctrl+Shift+P → "Calva: Start a Project REPL and Connect"
3. เลือก "Leiningen" (หรือ "deps.edn")
4. รอ REPL connect
```

### การ Evaluate Code ใน Calva

```clojure
;; ใน editor พิมพ์:
(defn hello [name]
  (str "Hello, " name))

;; วาง cursor ไว้ใน expression
;; Ctrl+Enter = evaluate current expression
;; Alt+Enter = evaluate top-level form

;; Result แสดงข้างๆ ทันที (inline evaluation)
(hello "World")  ; => "Hello, World!"

;; Evaluate ทั้งไฟล์
;; Ctrl+Alt+C, N = evaluate namespace
```

---

## ขั้นตอนที่ 55: Paredit - Structural Editing

Paredit เป็น plugin ที่ช่วยให้แก้ไข S-expressions ได้อย่างมีโครงสร้าง

```
ปัญหา: ใน Lisp ถ้าวงเล็บไม่สมดุล code พัง
วิธีแก้: ใช้ Paredit ที่รับประกันว่าวงเล็บสมดุลเสมอ
```

### Paredit Commands (ใน Calva)

```
Tab                = Indent code correctly
Ctrl+Right         = Slurp (ดึง element ถัดไปเข้ามา)
Ctrl+Left          = Barf (ไล่ element ออกไป)
Ctrl+Shift+S       = Splice (ลบวงเล็บรอบๆ)
Alt+Up             = Splice and kill backward
Alt+Down           = Wrap selection in ()

;; ตัวอย่าง Slurp:
;; ก่อน: (foo bar) baz
;; Slurp Right: (foo bar baz)

;; ตัวอย่าง Barf:
;; ก่อน: (foo bar baz)
;; Barf Right: (foo bar) baz
```

---

## ขั้นตอนที่ 56: REPL history และ Shortcuts

```bash
# Leiningen REPL shortcuts:
# Up/Down Arrow    = History navigation
# Ctrl+C           = Interrupt
# Ctrl+D           = Exit
# Tab              = Autocomplete

# Rebel Readline (enhanced REPL) shortcuts:
# Ctrl+R           = Search history
# Ctrl+A           = Go to beginning of line
# Ctrl+E           = Go to end of line
# Alt+Backspace    = Delete word backward
# Alt+D            = Delete word forward
```

```clojure
;; Save REPL history manually
(spit "repl-history.clj"
  (with-out-str
    (println ";; REPL Session" (java.util.Date.))
    (println)
    ;; ... history ...
    ))
```

---

## ขั้นตอนที่ 57: เชื่อมต่อกับ Running Application

```clojure
;; ใน production code เปิด nREPL server
;; project.clj
{:profiles {:dev {:dependencies [[nrepl/nrepl "1.3.0"]]}}}

;; ใน -main หรือ start function
(require '[nrepl.server :as nrepl])

(def nrepl-server
  (nrepl/start-server :port 7888))

;; เชื่อมจาก editor ไปที่ localhost:7888
;; แล้วสามารถ inspect/modify running application ได้!
```

---

## ขั้นตอนที่ 58: Tap และ Debug Middleware

```clojure
;; tap> system (Clojure 1.10+)
;; ส่งข้อมูลไปที่ tap listener

;; Register listener
(add-tap println)
;; ทุก tap> จะ println

(tap> {:event :user-login :user "สมชาย"})
;; {:event :user-login, :user "สมชาย"}

(tap> [1 2 3 4 5])
;; [1 2 3 4 5]

;; เอา listener ออก
(remove-tap println)

;; ใช้กับ Portal
(require '[portal.api :as p])
(def portal-instance (p/open))
(add-tap #'p/submit)

;; ทุก tap> จะแสดงใน Portal browser!
(tap> {:complex :data :nested {:a 1 :b [1 2 3]}})
```

---

## ขั้นตอนที่ 59: สรุป REPL Best Practices

```
1. REPL First, File Second
   - ทดลองใน REPL ก่อนเสมอ
   - เมื่อพอใจ save ลงไฟล์

2. Rich Comment Blocks
   - ใส่ example calls ใน (comment ...) blocks
   - ช่วยทั้ง documentation และ testing

3. Small Steps
   - ทดสอบทีละขั้นตอนเล็กๆ
   - อย่า compile ทั้ง feature พร้อมกัน

4. Data First
   - สร้าง sample data ก่อน
   - ทดสอบ transformations กับ data จริง

5. Use doc/source/apropos
   - อย่า google ทุกอย่าง
   - ดูจาก REPL โดยตรง

6. Reload mindfully
   - ใช้ :reload เมื่อแก้ไขไฟล์
   - ใช้ :reload-all เมื่อเปลี่ยน dependencies

7. Keep REPL state clean
   - restart บ้างเพื่อ clean state
   - อย่าพึ่งพา REPL state ใน production code
```

---

## ขั้นตอนที่ 60: Project Exercise - REPL Exploration

### Exercise: ค้นพบ Clojure.core ด้วย REPL

```clojure
;; ทำใน REPL:

;; 1. นับ functions ใน clojure.core
(count (ns-publics 'clojure.core))
;; => 600+ functions!

;; 2. ดู functions ที่เกี่ยวกับ sequence
(filter #(re-find #"seq" (name %))
        (keys (ns-publics 'clojure.core)))

;; 3. ค้นหา functions ที่รับ predicate
(find-doc "predicate")

;; 4. สำรวจ type hierarchy
(ancestors (type []))
(ancestors (type {}))
(ancestors (type #{}))

;; 5. เปรียบเทียบ performance
(time (apply + (range 1000000)))
(time (reduce + (range 1000000)))
(time (transduce identity + (range 1000000)))

;; 6. สำรวจ lazy sequences
(take 10 (iterate inc 0))
(take 10 (cycle [1 2 3]))
(take 10 (repeat "hello"))

;; 7. ทดลอง function composition
(def transform
  (comp
   (partial filter odd?)
   (partial map #(* % %))
   (partial filter #(> % 5))))

(transform (range 20))
```

### Project: Interactive Data Explorer

```clojure
;; สร้าง mini data explorer ใน REPL

(def sample-dataset
  (for [i (range 100)]
    {:id i
     :name (str "User-" i)
     :score (rand-int 100)
     :department (rand-nth ["Engineering" "Marketing" "Sales" "HR"])}))

;; Functions สำหรับ explore
(defn summary [data]
  {:count (count data)
   :avg-score (/ (reduce + (map :score data)) (count data))
   :departments (frequencies (map :department data))})

(defn top-scorers [data n]
  (->> data
       (sort-by :score >)
       (take n)
       (map #(select-keys % [:name :score]))))

(defn by-department [data dept]
  (filter #(= dept (:department %)) data))

;; ทดลองใน REPL:
(summary sample-dataset)
(top-scorers sample-dataset 5)
(by-department sample-dataset "Engineering")
(summary (by-department sample-dataset "Engineering"))
```

---

### อ่านต่อใน Part 3: Types พื้นฐาน และ Collections →

---

*Part 2 จาก 100+ | ขั้นตอน 31-60 จาก 1000+*
