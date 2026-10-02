# Part 3: Types พื้นฐาน และ Collections
## ขั้นตอนที่ 61-90: Mastering Clojure Data Structures

---

## บทนำ

Collections ใน Clojure ไม่เหมือนกับภาษาอื่น เพราะ:
1. **Immutable** - ไม่เปลี่ยนแปลงหลังสร้าง
2. **Persistent** - Share structure กับ version เก่า (efficient!)
3. **Polymorphic** - Functions เดียวกันใช้กับทุก collection
4. **Lazy** - คำนวณเมื่อต้องการเท่านั้น

---

## ขั้นตอนที่ 61: Vector - Ordered, Indexed Collection

```clojure
;; สร้าง Vector
[]                        ; vector ว่าง
[1 2 3]                  ; vector ของตัวเลข
["a" "b" "c"]            ; vector ของ strings
[1 "hello" :key true]    ; mixed types
[[1 2] [3 4]]            ; nested vectors

;; สร้างด้วย functions
(vector 1 2 3)            ; => [1 2 3]
(vec (range 5))           ; => [0 1 2 3 4]
(vec '(1 2 3))            ; => [1 2 3]

;; Access Elements
(def v [10 20 30 40 50])

(nth v 0)                 ; => 10 (first element)
(nth v 2)                 ; => 30
(nth v 10 :not-found)    ; => :not-found (default value)

(first v)                 ; => 10
(second v)               ; => 20
(last v)                  ; => 50

(v 0)                     ; => 10 (vector เป็น function!)
(v 3)                     ; => 40

;; Counting
(count v)                 ; => 5
(empty? [])               ; => true
(empty? v)                ; => false

;; Adding Elements
(conj v 60)               ; => [10 20 30 40 50 60]
(conj v 5 6 7)            ; => [10 20 30 40 50 5 6 7]

;; Modifying (returns new vector)
(assoc v 2 99)            ; => [10 20 99 40 50]
(assoc v 0 100 4 500)    ; => [100 20 30 40 500]

;; Slicing
(subvec v 1 3)            ; => [20 30] (index 1 to 3, exclusive)
(subvec v 2)              ; => [30 40 50] (from index 2)

;; Searching
(contains? v 2)           ; => true (checks INDEX, not value!)
(.indexOf v 30)           ; => 2 (Java method, checks VALUE)
(some #(= % 30) v)       ; => true (Clojure way)
```

---

## ขั้นตอนที่ 62: List - Sequential, Prepend-Optimized

```clojure
;; สร้าง List
'()                       ; empty list (quote ป้องกัน evaluation)
'(1 2 3)                  ; quoted list = data
(list 1 2 3)             ; => (1 2 3)
(list "a" "b" "c")       ; => ("a" "b" "c")

;; List vs Vector
;; List: efficient prepend, สำหรับ code (เป็น S-expressions)
;; Vector: efficient random access, สำหรับ data

;; Operations
(def lst '(1 2 3 4 5))

(first lst)               ; => 1
(rest lst)                ; => (2 3 4 5)
(next lst)                ; => (2 3 4 5)
(last lst)                ; => 5 (O(n) - ช้า!)

;; Prepend (fast for lists)
(cons 0 lst)              ; => (0 1 2 3 4 5)
(conj lst 0)              ; => (0 1 2 3 4 5) prepend ใน list!

;; ต่างจาก Vector:
(conj [1 2 3] 4)         ; => [1 2 3 4] append
(conj '(1 2 3) 0)        ; => (0 1 2 3) prepend!

;; Convert
(vec lst)                 ; => [1 2 3 4 5]
(into [] lst)             ; => [1 2 3 4 5]

;; Checking
(list? '(1 2 3))          ; => true
(list? [1 2 3])            ; => false
(seq? '(1 2 3))            ; => true
(seq? [1 2 3])             ; => true (vectors are seqs too!)
```

---

## ขั้นตอนที่ 63: Map - Key-Value Pairs

```clojure
;; สร้าง Map
{}                          ; empty map
{:name "สมชาย" :age 25}    ; keyword keys
{"name" "สมชาย" "age" 25}  ; string keys
{1 "one" 2 "two"}          ; integer keys
{:mixed 1 "key" 2 42 :three} ; mixed keys

;; Nested maps
{:user {:name "สมชาย"
        :address {:city "กรุงเทพ"
                  :zip "10100"}}}

;; Operations
(def user {:name "สมชาย" :age 25 :email "somchai@example.com"})

;; Get values
(get user :name)            ; => "สมชาย"
(get user :phone)           ; => nil
(get user :phone "N/A")    ; => "N/A" (default)

;; Keyword เป็น function!
(:name user)                ; => "สมชาย"  
(:phone user)               ; => nil
(:phone user "N/A")        ; => "N/A"

;; Map เป็น function ด้วย!
(user :name)                ; => "สมชาย"

;; Add/Update (returns new map)
(assoc user :phone "0891234567")
;; => {:name "สมชาย", :age 25, :email "...", :phone "0891234567"}

(assoc user :age 26 :city "กรุงเทพ")
;; => updated map

;; Remove key
(dissoc user :email)
;; => {:name "สมชาย", :age 25}

(dissoc user :age :email)
;; => {:name "สมชาย"}

;; Update value
(update user :age inc)
;; => {:name "สมชาย", :age 26, ...}

(update user :name str " Jr.")
;; => {:name "สมชาย Jr.", ...}

;; Merge maps
(merge {:a 1 :b 2} {:b 3 :c 4})
;; => {:a 1, :b 3, :c 4}  (second map wins for duplicates)

(merge user {:age 26 :city "Bangkok"})

;; merge-with - custom merge fn
(merge-with + {:a 1 :b 2} {:a 3 :b 4})
;; => {:a 4, :b 6}

;; Keys and Values
(keys user)                 ; => (:name :age :email)
(vals user)                 ; => ("สมชาย" 25 "somchai@example.com")

;; Checking
(contains? user :name)      ; => true
(contains? user :phone)     ; => false
(map? user)                 ; => true
(count user)                ; => 3
```

---

## ขั้นตอนที่ 64: Nested Map Operations

```clojure
;; Nested maps
(def profile
  {:user {:name "สมชาย"
          :age 25
          :address {:city "กรุงเทพ"
                    :zip "10100"
                    :country "Thailand"}}
   :settings {:theme "dark" :language "th"}})

;; Access nested
(get-in profile [:user :name])
;; => "สมชาย"

(get-in profile [:user :address :city])
;; => "กรุงเทพ"

(get-in profile [:user :phone] "N/A")
;; => "N/A"

;; Update nested
(assoc-in profile [:user :age] 26)
;; returns profile with age updated

(assoc-in profile [:user :address :district] "Watthana")
;; adds new key

(update-in profile [:user :age] inc)
;; increments age

(update-in profile [:settings :theme]
           #(if (= % "dark") "light" "dark"))
;; toggles theme

;; เข้าถึงด้วย -> threading
(-> profile :user :address :city)
;; => "กรุงเทพ"

;; เปรียบเทียบ
;; Python: profile["user"]["address"]["city"]
;; Java:   profile.getUser().getAddress().getCity()
;; Clojure: (get-in profile [:user :address :city])
;; Clojure: (-> profile :user :address :city)
```

---

## ขั้นตอนที่ 65: Set - Unique Values

```clojure
;; สร้าง Set
#{}                         ; empty set
#{1 2 3}                   ; set ของตัวเลข
#{"a" "b" "c"}             ; set ของ strings
#{:foo :bar :baz}          ; set ของ keywords

;; ถ้ามีซ้ำจะถูก deduplicate อัตโนมัติ!
#{1 2 2 3 3 3}             ; => #{1 2 3}

;; สร้างจาก collections
(set [1 2 2 3 3])          ; => #{1 2 3}
(set "hello")              ; => #{\h \e \l \o}  (unique chars!)
(into #{} [1 2 2 3])       ; => #{1 2 3}

;; Checking membership
(contains? #{1 2 3} 2)     ; => true
(contains? #{1 2 3} 5)     ; => false
(#{1 2 3} 2)               ; => 2 (set เป็น function!)
(#{1 2 3} 5)               ; => nil

;; Set Operations (ต้อง require)
(require '[clojure.set :as set])

(def s1 #{1 2 3 4 5})
(def s2 #{3 4 5 6 7})

;; Union
(set/union s1 s2)
;; => #{1 2 3 4 5 6 7}

;; Intersection
(set/intersection s1 s2)
;; => #{3 4 5}

;; Difference
(set/difference s1 s2)
;; => #{1 2}

(set/difference s2 s1)
;; => #{6 7}

;; Subset/Superset
(set/subset? #{1 2} #{1 2 3})
;; => true

(set/superset? #{1 2 3} #{1 2})
;; => true

;; Adding/Removing
(conj #{1 2 3} 4)          ; => #{1 2 3 4}
(disj #{1 2 3} 2)          ; => #{1 3}
(disj #{1 2 3} 2 3)        ; => #{1}
```

---

## ขั้นตอนที่ 66: Sequences และ Seq Protocol

ทุก collection ใน Clojure implement `ISeq` protocol ทำให้ functions เดียวกันใช้ได้กับทุก collection

```clojure
;; seq - แปลง collection เป็น sequence
(seq [1 2 3])               ; => (1 2 3)
(seq {:a 1 :b 2})          ; => ([:a 1] [:b 2])
(seq "hello")              ; => (\h \e \l \l \o)
(seq #{1 2 3})             ; => (1 2 3) (order may vary)
(seq [])                   ; => nil (empty = nil!)
(seq nil)                  ; => nil

;; ตรวจสอบว่า empty
(empty? [])                 ; => true
(empty? nil)               ; => true
(nil? (seq []))            ; => true

;; seq? - ตรวจสอบว่าเป็น seq
(seq? '(1 2 3))            ; => true
(seq? [1 2 3])             ; => false (แต่สามารถทำเป็น seq ได้)
(seq? (seq [1 2 3]))       ; => true

;; seqable? - ตรวจสอบว่าสามารถทำเป็น seq ได้
(seqable? [1 2 3])         ; => true
(seqable? {:a 1})          ; => true
(seqable? "hello")         ; => true
(seqable? 42)              ; => false

;; ทุก sequence operation ใช้ได้กับทุก collection!
(first {:a 1 :b 2})        ; => [:a 1]
(rest {:a 1 :b 2})         ; => ([:b 2])
(count {:a 1 :b 2})        ; => 2
(map first {:a 1 :b 2})   ; => (:a :b)
```

---

## ขั้นตอนที่ 67: Core Sequence Functions - map, filter, reduce

```clojure
;; map - แปลงแต่ละ element
(map inc [1 2 3 4 5])
;; => (2 3 4 5 6)

(map str [1 2 3])
;; => ("1" "2" "3")

(map #(* % %) [1 2 3 4 5])
;; => (1 4 9 16 25)

;; map กับ multiple collections
(map + [1 2 3] [10 20 30])
;; => (11 22 33)

(map vector [1 2 3] ["a" "b" "c"])
;; => ([1 "a"] [2 "b"] [3 "c"])

;; filter - เลือก element
(filter even? [1 2 3 4 5 6])
;; => (2 4 6)

(filter #(> % 3) [1 2 3 4 5])
;; => (4 5)

(filter string? [1 "hello" :key 2 "world"])
;; => ("hello" "world")

;; remove - ตรงข้าม filter
(remove even? [1 2 3 4 5 6])
;; => (1 3 5)

;; reduce - รวม elements เป็นค่าเดียว
(reduce + [1 2 3 4 5])
;; => 15

(reduce * [1 2 3 4 5])
;; => 120

;; reduce กับ initial value
(reduce + 10 [1 2 3])
;; => 16

;; reduce สร้าง map จาก vector
(reduce (fn [m item]
          (assoc m (:id item) item))
        {}
        [{:id 1 :name "a"} {:id 2 :name "b"}])
;; => {1 {:id 1, :name "a"}, 2 {:id 2, :name "b"}}
```

---

## ขั้นตอนที่ 68: More Sequence Functions

```clojure
;; take/drop
(take 3 [1 2 3 4 5])       ; => (1 2 3)
(drop 3 [1 2 3 4 5])       ; => (4 5)
(take-last 3 [1 2 3 4 5])  ; => (3 4 5)
(drop-last 3 [1 2 3 4 5])  ; => (1 2)

;; take-while/drop-while
(take-while #(< % 4) [1 2 3 4 5])
;; => (1 2 3)

(drop-while #(< % 4) [1 2 3 4 5])
;; => (4 5)

;; split-at / split-with
(split-at 3 [1 2 3 4 5])
;; => [(1 2 3) (4 5)]

(split-with #(< % 4) [1 2 3 4 5])
;; => [(1 2 3) (4 5)]

;; partition
(partition 2 [1 2 3 4 5 6])
;; => ((1 2) (3 4) (5 6))

(partition 3 [1 2 3 4 5 6 7 8])
;; => ((1 2 3) (4 5 6))  7 8 ถูกตัดทิ้ง!

(partition 3 3 [:x] [1 2 3 4 5 6 7 8])
;; => ((1 2 3) (4 5 6) (7 8 :x))  padding!

(partition-all 3 [1 2 3 4 5 6 7 8])
;; => ((1 2 3) (4 5 6) (7 8))  ไม่ตัดทิ้ง!

;; flatten
(flatten [[1 2] [3 [4 5]] 6])
;; => (1 2 3 4 5 6)

;; concat
(concat [1 2 3] [4 5 6])
;; => (1 2 3 4 5 6)

(concat [1 2] [3 4] [5 6])
;; => (1 2 3 4 5 6)

;; interleave / interpose
(interleave [1 2 3] ["a" "b" "c"])
;; => (1 "a" 2 "b" 3 "c")

(interpose ", " ["a" "b" "c"])
;; => ("a" ", " "b" ", " "c")

(apply str (interpose ", " ["a" "b" "c"]))
;; => "a, b, c"
```

---

## ขั้นตอนที่ 69: Sort และ Group

```clojure
;; sort
(sort [3 1 4 1 5 9 2 6])
;; => (1 1 2 3 4 5 6 9)

(sort > [3 1 4 1 5 9])
;; => (9 5 4 3 1 1)

;; sort-by
(def users
  [{:name "Charlie" :age 25}
   {:name "Alice" :age 30}
   {:name "Bob" :age 22}])

(sort-by :name users)
;; => sorted by name alphabetically

(sort-by :age users)
;; => sorted by age ascending

(sort-by :age > users)
;; => sorted by age descending

(sort-by (juxt :age :name) users)
;; => sorted by age, then name

;; group-by
(group-by even? [1 2 3 4 5 6])
;; => {false (1 3 5), true (2 4 6)}

(def items [{:type :a :val 1}
            {:type :b :val 2}
            {:type :a :val 3}
            {:type :b :val 4}])

(group-by :type items)
;; => {:a [{:type :a, :val 1} {:type :a, :val 3}],
;;     :b [{:type :b, :val 2} {:type :b, :val 4}]}

;; frequencies
(frequencies ["a" "b" "a" "c" "b" "a"])
;; => {"a" 3, "b" 2, "c" 1}

(frequencies (map :type items))
;; => {:a 2, :b 2}
```

---

## ขั้นตอนที่ 70: Lazy Sequences

```clojure
;; Lazy sequences ไม่คำนวณจนกว่าจะถูก access

;; range - เป็น lazy
(range)           ; infinite sequence! ไม่ crash
(take 5 (range)) ; => (0 1 2 3 4)

(range 10)        ; => (0 1 2 3 4 5 6 7 8 9)
(range 5 10)      ; => (5 6 7 8 9)
(range 0 10 2)    ; => (0 2 4 6 8)

;; iterate - infinite lazy sequence
(take 10 (iterate inc 0))
;; => (0 1 2 3 4 5 6 7 8 9)

(take 5 (iterate #(* % 2) 1))
;; => (1 2 4 8 16)

;; repeat
(take 5 (repeat "hello"))
;; => ("hello" "hello" "hello" "hello" "hello")

(repeat 5 "hello")
;; => ("hello" "hello" "hello" "hello" "hello")

;; cycle - infinite repeating
(take 7 (cycle [1 2 3]))
;; => (1 2 3 1 2 3 1)

;; lazy-seq - สร้าง lazy sequence เอง
(defn natural-numbers []
  (lazy-seq
    (cons 1 (map inc (natural-numbers)))))

(take 5 (natural-numbers))
;; => (1 2 3 4 5)

;; Fibonacci ด้วย lazy-seq
(def fibs
  (lazy-seq
    (concat [0 1]
            (map + fibs (rest fibs)))))

(take 10 fibs)
;; => (0 1 1 2 3 5 8 13 21 34)

;; ระวัง! อย่า realize infinite sequences
;; (count (range))   ; ค้างตลอดไป!
;; (last (range))    ; ค้างตลอดไป!
```

---

## ขั้นตอนที่ 71: Threading Macros (สำคัญมาก!)

Threading macros ทำให้โค้ดอ่านง่ายขึ้นมาก

```clojure
;; ปัญหา: nested function calls อ่านยาก
(str/join ", " (map :name (filter #(> (:age %) 25) users)))
;; อ่านจากในออกนอก ยากมาก!

;; -> (Thread-first: แทรก argument ที่ position 1)
(-> users
    (filter #(> (:age %) 25))  ; ปัญหา: filter ต้องการ coll เป็น arg ที่ 2!
    (map :name)
    (str/join ", "))

;; ->> (Thread-last: แทรก argument ที่ position สุดท้าย)
(->> users
     (filter #(> (:age %) 25))
     (map :name)
     (str/join ", "))
;; อ่านง่ายกว่ามาก! บนลงล่าง

;; -> vs ->> เมื่อไหร่ใช้อะไร?
;; -> ใช้กับ data-first functions: assoc, update, get, (:keyword)
;; ->> ใช้กับ sequence functions: map, filter, reduce

;; ตัวอย่าง ->
(-> person
    :address
    :city
    str/upper-case)
;; = (str/upper-case (:city (:address person)))

;; ตัวอย่าง ->>
(->> numbers
     (filter even?)
     (map #(* % %))
     (reduce +))
;; = (reduce + (map #(* % %) (filter even? numbers)))

;; as-> (flexible threading)
(as-> value v
  (* v 2)
  (str "Result: " v)
  (str/upper-case v))

;; cond-> (conditional threading)
(cond-> user
  (nil? (:email user)) (assoc :email "default@example.com")
  (< (:age user) 18)   (assoc :is-minor true)
  true                  (assoc :processed true))

;; some-> (stop if nil)
(some-> user
        :address
        :city
        str/upper-case)
;; ถ้า user หรือ :address หรือ :city เป็น nil จะคืน nil
;; แทนที่จะ throw NullPointerException
```

---

## ขั้นตอนที่ 72: Collection Transformations

```clojure
;; into - เพิ่ม elements จาก collection หนึ่งไปอีกอัน
(into [] '(1 2 3))           ; => [1 2 3]
(into #{} [1 2 2 3])         ; => #{1 2 3}
(into {} [[:a 1] [:b 2]])   ; => {:a 1, :b 2}
(into [4 5 6] [1 2 3])      ; => [4 5 6 1 2 3]

;; mapv, filterv - returns vector (not lazy seq)
(mapv inc [1 2 3])           ; => [1 2 3 4] (vector)
(filterv even? [1 2 3 4])   ; => [2 4] (vector)

;; keep - map แล้ว remove nils
(keep #(when (even? %) (* % %)) [1 2 3 4 5 6])
;; => (4 16 36)

;; keep-indexed
(keep-indexed #(when (even? %1) %2) ["a" "b" "c" "d"])
;; => ("a" "c")

;; map-indexed
(map-indexed #(str %1 ":" %2) ["a" "b" "c"])
;; => ("0:a" "1:b" "2:c")

;; reduce-kv - reduce สำหรับ maps
(reduce-kv (fn [acc k v] (assoc acc k (inc v)))
           {}
           {:a 1 :b 2 :c 3})
;; => {:a 2, :b 3, :c 4}

;; update-vals (Clojure 1.11+)
(update-vals {:a 1 :b 2 :c 3} inc)
;; => {:a 2, :b 3, :c 4}

;; update-keys (Clojure 1.11+)
(update-keys {:a 1 :b 2} name)
;; => {"a" 1, "b" 2}
```

---

## ขั้นตอนที่ 73: Aggregation Functions

```clojure
;; sum
(reduce + [1 2 3 4 5])      ; => 15
(apply + [1 2 3 4 5])       ; => 15 (อาจ stack overflow กับ list ใหญ่!)

;; Better:
(transduce identity + [1 2 3 4 5])  ; => 15

;; min/max
(apply min [3 1 4 1 5 9])   ; => 1
(apply max [3 1 4 1 5 9])   ; => 9

;; count
(count [1 2 3])              ; => 3

;; any? / every? / not-any? / not-every?
(every? even? [2 4 6])      ; => true
(every? even? [2 4 5])      ; => false
(some even? [1 3 4 5])      ; => true (returns truthy value)
(some even? [1 3 5])        ; => nil (returns nil if not found)
(not-any? even? [1 3 5])    ; => true
(not-every? even? [1 2 3])  ; => true

;; distinct
(distinct [1 2 1 3 2 4])    ; => (1 2 3 4)

;; dedupe - ลบ consecutive duplicates
(dedupe [1 1 2 2 3 1 1])    ; => (1 2 3 1)

;; flatten
(flatten [[1 2] [3 [4 5]]])  ; => (1 2 3 4 5)

;; frequencies
(frequencies "abracadabra")
;; => {\a 5, \b 2, \r 2, \c 1, \d 1}

;; reductions - intermediate reduce results
(reductions + [1 2 3 4 5])
;; => (1 3 6 10 15)
```

---

## ขั้นตอนที่ 74: Destructuring - Collections

Destructuring ทำให้แยก collection ออกเป็น variables ง่ายขึ้น

```clojure
;; Vector destructuring
(let [[a b c] [1 2 3]]
  (println a b c))
;; => 1 2 3

(let [[first second & rest] [1 2 3 4 5]]
  (println first second rest))
;; => 1 2 (3 4 5)

;; Skip elements ด้วย _
(let [[_ _ third] [1 2 3 4 5]]
  third)
;; => 3

;; Map destructuring
(let [{name :name age :age} {:name "สมชาย" :age 25}]
  (println name age))
;; => สมชาย 25

;; ชื่อตรงกับ key ใช้ :keys
(let [{:keys [name age email]} {:name "สมชาย" :age 25 :email "test@test.com"}]
  (println name age email))
;; => สมชาย 25 test@test.com

;; Default values
(let [{:keys [name age role]
       :or {role "user" age 0}} {:name "สมชาย"}]
  (println name age role))
;; => สมชาย 0 user

;; Nested destructuring
(let [{name :name
       {city :city} :address}
      {:name "สมชาย" :address {:city "กรุงเทพ"}}]
  (println name city))
;; => สมชาย กรุงเทพ

;; In function parameters
(defn greet [{:keys [name age] :or {age 0}}]
  (str "สวัสดี " name " อายุ " age " ปี"))

(greet {:name "สมชาย" :age 25})
;; => "สวัสดี สมชาย อายุ 25 ปี"

(greet {:name "สมหญิง"})
;; => "สวัสดี สมหญิง อายุ 0 ปี"
```

---

## ขั้นตอนที่ 75: Keywords และ Namespaced Keywords

```clojure
;; Keywords เป็น identifiers ที่ evaluate ตัวเอง
:name                   ; => :name
:user/name              ; namespaced keyword

;; Keywords เป็น functions
(:name {:name "สมชาย"})  ; => "สมชาย"
(:age {:name "สมชาย"} 0) ; => 0 (default)

;; Namespaced keywords (ดีกว่า global keywords)
(def user {:user/name "สมชาย"
           :user/age 25
           :user/email "test@test.com"})

(:user/name user)        ; => "สมชาย"

;; Auto-resolve namespace
(def ns-user #:user{:name "สมชาย"
                    :age 25})
;; = {:user/name "สมชาย", :user/age 25}

;; ::keyword = auto namespace ด้วย current namespace
(in-ns 'my.app.user)
::name              ; => :my.app.user/name
::id                ; => :my.app.user/id

;; Useful สำหรับ Spec และ databases
;; (ป้องกัน key collision ข้ามระบบ)
```

---

## ขั้นตอนที่ 76: Transforming Complex Data

```clojure
;; ตัวอย่างจริง: Process API Response
(def api-response
  {:status "success"
   :data {:users [{:id 1 :first_name "สมชาย" :last_name "ใจดี" :age 25 :active true}
                  {:id 2 :first_name "สมหญิง" :last_name "ใจงาม" :age 30 :active false}
                  {:id 3 :first_name "มานะ" :last_name "ขยัน" :age 22 :active true}]
          :total 3}})

;; ดึง users ที่ active และแปลงรูปแบบ
(->> (get-in api-response [:data :users])
     (filter :active)
     (map (fn [{:keys [id first_name last_name age]}]
            {:id id
             :full-name (str first_name " " last_name)
             :age age
             :is-adult (>= age 18)}))
     (sort-by :age))

;; => ({:id 3, :full-name "มานะ ขยัน", :age 22, :is-adult true}
;;     {:id 1, :full-name "สมชาย ใจดี", :age 25, :is-adult true})
```

---

## ขั้นตอนที่ 77: Zippers - Tree Manipulation

```clojure
(require '[clojure.zip :as zip])

;; สร้าง zipper จาก vector (tree)
(def tree [1 [2 3] [4 [5 6]]])
(def z (zip/vector-zip tree))

;; Navigate
(zip/node z)         ; => [1 [2 3] [4 [5 6]]]  (root)
(zip/down z)         ; => zipper at 1
(-> z zip/down zip/right)     ; => zipper at [2 3]
(-> z zip/down zip/right zip/node) ; => [2 3]

;; Edit
(-> z
    zip/down
    (zip/replace 99)   ; replace 1 with 99
    zip/root)          ; => [99 [2 3] [4 [5 6]]]

;; Walk tree
(zip/xml-zip {:tag :root
              :content [{:tag :item :content ["a"]}
                        {:tag :item :content ["b"]}]})
```

---

## ขั้นตอนที่ 78: clojure.walk

```clojure
(require '[clojure.walk :as walk])

;; postwalk - traverse จากล่างขึ้นบน
(walk/postwalk
  (fn [node]
    (if (number? node)
      (* node 2)
      node))
  {:a 1 :b {:c 2 :d [3 4 5]}})
;; => {:a 2, :b {:c 4, :d [6 8 10]}}

;; prewalk - traverse จากบนลงล่าง
(walk/prewalk
  (fn [node]
    (if (map? node)
      (assoc node :processed true)
      node))
  {:a 1 :b {:c 2}})
;; => {:a 1, :b {:c 2, :processed true}, :processed true}

;; keywordize-keys - แปลง string keys เป็น keywords
(walk/keywordize-keys {"name" "สมชาย" "age" 25 "nested" {"key" "val"}})
;; => {:name "สมชาย", :age 25, :nested {:key "val"}}

;; stringify-keys - ตรงข้าม
(walk/stringify-keys {:name "สมชาย" :age 25})
;; => {"name" "สมชาย", "age" 25}
```

---

## ขั้นตอนที่ 79: Performance ของ Collections

```clojure
;; Complexity ของแต่ละ operation:
;;
;; Vector:
;;   conj      O(1) amortized
;;   nth       O(1)
;;   assoc     O(log32 n)
;;   count     O(1)
;;
;; List:
;;   conj      O(1) (prepend)
;;   first     O(1)
;;   rest      O(1)
;;   nth       O(n)  ← ช้า!
;;   count     O(n)  ← ช้า! (use counted? to check)
;;
;; HashMap:
;;   assoc     O(log32 n)
;;   get       O(log32 n)
;;   dissoc    O(log32 n)
;;   count     O(1)
;;
;; HashSet:
;;   conj      O(log32 n)
;;   contains? O(log32 n)
;;   disj      O(log32 n)

;; ตัวอย่างการเลือก collection ที่เหมาะสม:

;; ต้องการ random access → Vector
(def indexed-data (vec (range 1000000)))
(indexed-data 500000)  ; O(1)

;; ต้องการ unique + lookup → Set
(def unique-ids (into #{} (range 1000000)))
(contains? unique-ids 500000)  ; O(log32 n)

;; ต้องการ key-value lookup → Map
(def user-by-id (zipmap (range 1000) 
                        (repeatedly 1000 #(hash-map :name "x"))))
(get user-by-id 500)  ; O(log32 n)

;; Stack (LIFO) → List หรือ Vector
(def stack [])
(def stack (conj stack 1 2 3))  ; [1 2 3] push
(peek stack)                     ; 3 (top)
(pop stack)                      ; [1 2] (remove top)

;; Queue (FIFO) → PersistentQueue
(def q clojure.lang.PersistentQueue/EMPTY)
(def q (conj q 1 2 3))     ; enqueue
(peek q)                    ; 1 (front)
(pop q)                     ; remove front
```

---

## ขั้นตอนที่ 80: Transients - Mutable for Performance

เมื่อต้องการ performance สูง สามารถใช้ transient collections ได้

```clojure
;; สร้าง transient (mutable copy)
(def tv (transient [1 2 3]))

;; Mutate operations
(conj! tv 4)      ; adds 4, returns tv
(assoc! tv 0 99)  ; changes index 0
(pop! tv)         ; removes last

;; กลับเป็น persistent
(persistent! tv)  ; => [99 2 3 4]

;; ระวัง! หลัง persistent! ไม่สามารถ mutate transient อีก!

;; Use case: building large collections
(defn build-vector [n]
  (loop [i 0
         result (transient [])]
    (if (= i n)
      (persistent! result)
      (recur (inc i) (conj! result i)))))

;; Performance comparison
(time (into [] (range 100000)))           ; ~10ms
(time (build-vector 100000))              ; ~5ms (เร็วกว่า 2x)
```

---

## ขั้นตอนที่ 81: Record - Named Map Type

```clojure
;; defrecord - map ที่มีชื่อและ schema
(defrecord Point [x y])

;; สร้าง instance
(def p (->Point 3 4))
(def p (Point. 3 4))  ; Java style

;; Access
(:x p)    ; => 3
(:y p)    ; => 4
(.x p)    ; => 3 (Java interop)

;; Records implement map interface
(assoc p :z 5)    ; => #my.ns.Point{:x 3, :y 4, :z 5}
(merge p {:color :red}) ; => #my.ns.Point{:x 3, :y 4, :color :red}
(keys p)          ; => (:x :y)
(map? p)          ; => true

;; ข้อดีของ Record:
;; 1. Named type → ใช้ dispatch ได้
;; 2. Fast field access (Java fields, ไม่ใช่ hash lookup)
;; 3. Implement protocols

;; ตัวอย่างจริง
(defrecord User [id name email created-at]
  Object
  (toString [this]
    (str "User[" id ":" name "]")))

(def user (->User 1 "สมชาย" "somchai@test.com" (java.util.Date.)))
(str user)   ; => "User[1:สมชาย]"
(type user)  ; => my.ns.User
```

---

## ขั้นตอนที่ 82: Real-World Example - Shopping Cart

```clojure
(ns shopping-cart.core)

;; Data model
(defn create-cart []
  {:items {} :discount 0})

(defn add-item
  "เพิ่ม item ใน cart"
  [cart product-id product-name price qty]
  (update-in cart [:items product-id]
             (fn [existing]
               (if existing
                 (update existing :qty + qty)
                 {:id product-id
                  :name product-name
                  :price price
                  :qty qty}))))

(defn remove-item
  "ลบ item ออกจาก cart"
  [cart product-id]
  (update cart :items dissoc product-id))

(defn update-qty
  "อัพเดท quantity"
  [cart product-id new-qty]
  (if (<= new-qty 0)
    (remove-item cart product-id)
    (assoc-in cart [:items product-id :qty] new-qty)))

(defn subtotal
  "คำนวณราคาก่อน discount"
  [cart]
  (->> (vals (:items cart))
       (map #(* (:price %) (:qty %)))
       (reduce + 0)))

(defn apply-discount
  "ใส่ discount %"
  [cart discount-pct]
  (assoc cart :discount discount-pct))

(defn total
  "ราคาสุดท้ายหลัง discount"
  [cart]
  (let [sub (subtotal cart)
        disc (* sub (/ (:discount cart) 100))]
    (- sub disc)))

(defn item-count [cart]
  (->> (vals (:items cart))
       (map :qty)
       (reduce + 0)))

(defn display-cart [cart]
  (println "\n=== Shopping Cart ===")
  (doseq [[_ item] (:items cart)]
    (printf "%-20s %5d x %8.2f = %10.2f%n"
            (:name item)
            (:qty item)
            (double (:price item))
            (double (* (:price item) (:qty item)))))
  (println "---")
  (printf "Subtotal: %10.2f%n" (double (subtotal cart)))
  (when (> (:discount cart) 0)
    (printf "Discount (%.0f%%): %6.2f%n"
            (double (:discount cart))
            (double (* (subtotal cart) (/ (:discount cart) 100)))))
  (printf "Total: %13.2f%n" (double (total cart)))
  (printf "Items: %d%n" (item-count cart)))

;; ทดสอบ
(comment
  (def cart (create-cart))
  (def cart (add-item cart "p001" "iPhone 15" 45000 1))
  (def cart (add-item cart "p002" "AirPods Pro" 9000 2))
  (def cart (add-item cart "p001" "iPhone 15" 45000 1))  ; เพิ่มอีก
  (def cart (apply-discount cart 10))

  (display-cart cart)
  ;; === Shopping Cart ===
  ;; iPhone 15                 2 x  45000.00 =  90000.00
  ;; AirPods Pro               2 x   9000.00 =  18000.00
  ;; ---
  ;; Subtotal:       108000.00
  ;; Discount (10%):  10800.00
  ;; Total:           97200.00
  ;; Items: 4
  )
```

---

## ขั้นตอนที่ 83: Collection Best Practices

```clojure
;; 1. ใช้ keywords เป็น map keys (ไม่ใช่ strings)
;; ❌ {"name" "สมชาย", "age" 25}
;; ✅ {:name "สมชาย", :age 25}

;; 2. ใช้ vector สำหรับ ordered data, set สำหรับ unique
;; ❌ (list 1 2 3)      ; ถ้าต้องการ index access
;; ✅ [1 2 3]            ; vector
;; ✅ #{:admin :user}    ; unique roles

;; 3. Destructure อย่างชัดเจน
;; ❌ (fn [user] (do-something (:name user) (:email user)))
;; ✅ (fn [{:keys [name email]}] (do-something name email))

;; 4. ใช้ threading macros แทน nesting
;; ❌ (str/join ", " (map :name (filter #(> (:age %) 18) users)))
;; ✅ (->> users
;;         (filter #(> (:age %) 18))
;;         (map :name)
;;         (str/join ", "))

;; 5. ใช้ update/update-in แทน get-then-assoc
;; ❌ (assoc m :count (inc (:count m)))
;; ✅ (update m :count inc)

;; 6. ใช้ get-in สำหรับ nested access
;; ❌ (:city (:address (:user data)))
;; ✅ (get-in data [:user :address :city])

;; 7. Prefer lazy sequences
;; แทนที่จะสร้าง collection ทั้งหมดก่อน
;; ใช้ lazy operations ซึ่งคำนวณ on-demand
```

---

## ขั้นตอนที่ 84: Advanced Map Operations

```clojure
;; select-keys
(select-keys {:a 1 :b 2 :c 3 :d 4} [:a :c])
;; => {:a 1, :c 3}

;; rename-keys
(require '[clojure.set :as set])
(set/rename-keys {:first_name "สมชาย" :last_name "ใจดี"}
                 {:first_name :firstName :last_name :lastName})
;; => {:firstName "สมชาย", :lastName "ใจดี"}

;; filter map entries
(into {} (filter #(string? (val %)) {:a "hello" :b 42 :c "world"}))
;; => {:a "hello", :c "world"}

;; map ทั้ง keys และ values
(into {} (map (fn [[k v]] [(name k) (* v 2)]) {:a 1 :b 2}))
;; => {"a" 2, "b" 4}

;; juxt - apply multiple functions
(def extract (juxt :name :age :email))
(extract {:name "สมชาย" :age 25 :email "test@test.com"})
;; => ["สมชาย" 25 "test@test.com"]

(map (juxt :name :age) users)
;; => (["สมชาย" 25] ["สมหญิง" 30] ...)

;; fnil - wrap function with nil handling
(def safe-inc (fnil inc 0))
(safe-inc nil)   ; => 1 (แทนที่จะ NullPointerException)
(safe-inc 5)     ; => 6

;; ใช้กับ update
(update {} :counter (fnil inc 0))
;; => {:counter 1}
```

---

## ขั้นตอนที่ 85: Sequence Generators

```clojure
;; repeatedly - call function ซ้ำๆ (lazy)
(take 5 (repeatedly #(rand-int 100)))
;; => (42 87 13 56 29)  (random!)

(take 5 (repeatedly (fn [] {:id (rand-int 1000) :active (rand-nth [true false])})))

;; iterate
(take 5 (iterate * 2))
;; ERROR: ต้องมี initial value
(take 5 (iterate #(* 2 %) 1))
;; => (1 2 4 8 16)

;; unfold pattern
(defn unfold [f init]
  (lazy-seq
    (when-let [[val next] (f init)]
      (cons val (unfold f next)))))

(take 5 (unfold (fn [n] [n (inc n)]) 0))
;; => (0 1 2 3 4)

;; generate Fibonacci
(defn fib-seq
  ([] (fib-seq 0 1))
  ([a b] (lazy-seq (cons a (fib-seq b (+ a b))))))

(take 10 (fib-seq))
;; => (0 1 1 2 3 5 8 13 21 34)
```

---

## ขั้นตอนที่ 86: Collection Utilities

```clojure
;; zipmap - สร้าง map จาก keys และ values
(zipmap [:a :b :c] [1 2 3])
;; => {:a 1, :b 2, :c 3}

;; zipmap กับ range
(zipmap (map :id users) users)
;; => {1 {:id 1 ...}, 2 {...}, ...}

;; frequencies + sort
(->> "hello world hello clojure"
     (re-seq #"\w+")
     frequencies
     (sort-by val >))
;; => (["hello" 2] ["world" 1] ["clojure" 1])

;; min-by / max-by
(apply min-key :age users)  ; user อายุน้อยสุด
(apply max-key :salary users)  ; user เงินเดือนสูงสุด

;; distinct-by (ไม่มีใน core ต้องเขียนเอง)
(defn distinct-by [key-fn coll]
  (->> coll
       (group-by key-fn)
       vals
       (map first)))

(distinct-by :department users)
;; user หนึ่งคนต่อหนึ่ง department

;; flatten nested maps
(defn flatten-map
  ([m] (flatten-map m []))
  ([m prefix]
   (reduce-kv
    (fn [acc k v]
      (let [full-key (if (seq prefix)
                       (keyword (str (name prefix) "." (name k)))
                       k)]
        (if (map? v)
          (merge acc (flatten-map v full-key))
          (assoc acc full-key v))))
    {}
    m)))

(flatten-map {:user {:name "สมชาย" :address {:city "กรุงเทพ"}}})
;; => {:user.name "สมชาย", :user.address.city "กรุงเทพ"}
```

---

## ขั้นตอนที่ 87: Working with Strings as Sequences

```clojure
;; Strings เป็น sequences ของ characters
(seq "hello")
;; => (\h \e \l \l \o)

;; map กับ string
(map char/upper-case "hello")
;; error! ต้อง cast

(map str "hello")
;; => ("h" "e" "l" "l" "o")

;; filter characters
(filter #(re-find #"[aeiou]" (str %)) "hello world")
;; => (\e \o \o)

;; count characters
(count "hello")          ; => 5
(count "สวัสดี")         ; => 6

;; สร้าง string จาก characters
(apply str (reverse "hello"))
;; => "olleh"

(apply str (filter #(not= % \space) "hello world"))
;; => "helloworld"

;; String partitioning
(partition 2 "abcdef")
;; => ((\a \b) (\c \d) (\e \f))

(map #(apply str %) (partition 2 "abcdef"))
;; => ("ab" "cd" "ef")
```

---

## ขั้นตอนที่ 88: Pattern Matching with core.match

```clojure
;; เพิ่ม dependency:
;; [org.clojure/core.match "1.0.0"]

(require '[clojure.core.match :refer [match]])

;; Simple pattern matching
(match [1 2]
  [1 2] "one two"
  [1 _] "one anything"
  :else "other")
;; => "one two"

;; Match maps
(match {:name "สมชาย" :age 25}
  {:age (:or 18 19 20)} "teenager"
  {:age age :name name} (str name " is " age " years old"))
;; => "สมชาย is 25 years old"

;; Match with guards
(match [3]
  [n] (cond
        (< n 0) "negative"
        (= n 0) "zero"
        :else "positive"))
;; => "positive"

;; Practical example: HTTP response handling
(defn handle-response [response]
  (match response
    {:status 200 :body body} {:success true :data body}
    {:status 404}             {:success false :error "Not found"}
    {:status 500 :message m} {:success false :error (str "Server error: " m)}
    _                         {:success false :error "Unknown error"}))
```

---

## ขั้นตอนที่ 89: Summary และ Collection Quick Reference

```
Collection  | Create    | Add        | Remove   | Access    | Use Case
------------|-----------|------------|----------|-----------|----------
Vector []   | [1 2 3]   | (conj v x) | (pop v)  | (v 0)     | Ordered, indexed
List ()     | '(1 2 3)  | (cons x l) | (rest l) | (first l) | Stack, code
Map {}      | {:a 1}    | (assoc m)  | (dissoc) | (:key m)  | Key-value
Set #{}     | #{1 2 3}  | (conj s x) | (disj s) | (s x)     | Unique values

Threading Macros:
->   : thread as FIRST argument  (maps, records)
->>  : thread as LAST argument   (sequences)
as-> : flexible threading
some-> : nil-safe threading
cond-> : conditional threading
```

---

## ขั้นตอนที่ 90: Project Exercise - Student Grade System

```clojure
(ns grade-system.core)

(def students
  [{:id 1 :name "สมชาย" :scores [85 90 78 92 88]}
   {:id 2 :name "สมหญิง" :scores [95 88 92 96 90]}
   {:id 3 :name "มานะ" :scores [70 65 72 68 75]}
   {:id 4 :name "มาลี" :scores [88 82 90 85 87]}
   {:id 5 :name "สมศักดิ์" :scores [55 60 58 62 65]}])

(defn average [scores]
  (/ (reduce + scores) (count scores)))

(defn grade [avg]
  (cond
    (>= avg 90) "A"
    (>= avg 80) "B"
    (>= avg 70) "C"
    (>= avg 60) "D"
    :else "F"))

(defn process-student [student]
  (let [avg (average (:scores student))]
    (assoc student
           :average (double avg)
           :grade (grade avg)
           :passed? (>= avg 60))))

(defn class-report [students]
  (let [processed (map process-student students)
        passed (filter :passed? processed)
        failed (remove :passed? processed)
        class-avg (average (map :average processed))]
    {:students (sort-by :average > processed)
     :class-average (double class-avg)
     :pass-rate (* 100 (/ (count passed) (count students)))
     :top-student (:name (first (sort-by :average > processed)))
     :failing-students (map :name failed)}))

;; ใน REPL:
(comment
  (def report (class-report students))
  
  ;; แสดงผล
  (println "=== Class Report ===")
  (doseq [s (:students report)]
    (printf "%s: avg=%.1f grade=%s %s%n"
            (:name s) (:average s) (:grade s)
            (if (:passed? s) "✓" "✗")))
  (printf "%nClass Average: %.1f%n" (:class-average report))
  (printf "Pass Rate: %.0f%%%n" (:pass-rate report))
  (println "Top Student:" (:top-student report))
  (when (seq (:failing-students report))
    (println "Failing:" (str/join ", " (:failing-students report)))))
```

---

### อ่านต่อใน Part 4: Functions ขั้นสูง →

---

*Part 3 จาก 100+ | ขั้นตอน 61-90 จาก 1000+*
