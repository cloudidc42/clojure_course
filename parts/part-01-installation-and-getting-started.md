# Part 1: ติดตั้งและเริ่มต้น Clojure
## ขั้นตอนที่ 1-30: Setup Environment และ Hello World

---

## บทนำ

ยินดีต้อนรับสู่โลกของ Clojure! ภาษาโปรแกรมที่จะเปลี่ยนวิธีคิดของคุณเกี่ยวกับการเขียนโค้ดไปตลอดกาล

Clojure เป็น dialect ของ Lisp ที่ทำงานบน JVM (Java Virtual Machine) ออกแบบโดย Rich Hickey ในปี 2007 ด้วยปรัชญาที่ว่า "Simple is not Easy" - การทำให้โค้ดเรียบง่าย ถูกต้อง และบำรุงรักษาง่าย

### ทำไม Clojure?

```
ปัญหาของโปรแกรมทั่วไป          |  วิธีที่ Clojure แก้ปัญหา
--------------------------------|--------------------------------
State mutation bugs              |  Immutable data by default
Concurrency hell                |  Software Transactional Memory
Verbose Java boilerplate        |  Concise, expressive syntax  
Hard to test                    |  Pure functions everywhere
Slow feedback loop              |  REPL-driven development
```

---

## ขั้นตอนที่ 1: ติดตั้ง Java Development Kit (JDK)

Clojure ทำงานบน JVM ดังนั้นต้องติดตั้ง Java ก่อน

### macOS

```bash
# ใช้ Homebrew (แนะนำ)
brew install openjdk@21

# หรือดาวน์โหลดจาก https://adoptium.net/

# ตรวจสอบการติดตั้ง
java -version
# output: openjdk version "21.0.x" ...

javac -version
# output: javac 21.0.x
```

### Windows

```powershell
# ใช้ winget (Windows Package Manager)
winget install Microsoft.OpenJDK.21

# หรือ Chocolatey
choco install openjdk21

# ตรวจสอบการติดตั้ง
java -version
```

### Linux (Ubuntu/Debian)

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install openjdk-21-jdk

# Fedora/RHEL
sudo dnf install java-21-openjdk-devel

# ตรวจสอบ
java -version
javac -version

# ตั้งค่า JAVA_HOME
echo 'export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64' >> ~/.bashrc
echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

---

## ขั้นตอนที่ 2: ติดตั้ง Clojure

### macOS

```bash
# ใช้ Homebrew
brew install clojure/tools/clojure

# ตรวจสอบ
clojure --version
# output: Clojure CLI version 1.12.x.x
```

### Windows

```powershell
# ดาวน์โหลด Windows installer จาก
# https://github.com/clojure/tools.deps/wiki/clj-on-Windows

# หรือใช้ PowerShell script
Invoke-Expression (New-Object System.Net.WebClient).DownloadString('https://download.clojure.org/install/win-install-1.12.0.1488.ps1')
```

### Linux

```bash
# Debian/Ubuntu
curl -L -O https://github.com/clojure/brew-install/releases/latest/download/linux-install.sh
chmod +x linux-install.sh
sudo ./linux-install.sh

# ตรวจสอบ
clojure --version
```

---

## ขั้นตอนที่ 3: ติดตั้ง Leiningen (Build Tool)

Leiningen เป็น build tool ที่นิยมใช้กับ Clojure มากที่สุด

### macOS

```bash
brew install leiningen

# ตรวจสอบ
lein version
# output: Leiningen 2.x.x on Java 21 ...
```

### Linux/macOS (Manual)

```bash
# ดาวน์โหลด script
curl -O https://raw.githubusercontent.com/technomancy/leiningen/stable/bin/lein
chmod +x lein
sudo mv lein /usr/local/bin/

# ครั้งแรกจะ download dependencies อัตโนมัติ
lein version
```

### Windows

```powershell
# ใช้ Chocolatey
choco install lein

# ตรวจสอบ
lein version
```

---

## ขั้นตอนที่ 4: ติดตั้ง Editor

### VS Code + Calva (แนะนำสำหรับมือใหม่)

```bash
# ติดตั้ง VS Code จาก https://code.visualstudio.com/

# ติดตั้ง Calva extension
# Extensions → ค้นหา "Calva" → Install

# Features:
# - Syntax highlighting
# - REPL integration  
# - Code evaluation inline
# - Paredit (structural editing)
# - Debugger
```

### IntelliJ IDEA + Cursive

```
1. ติดตั้ง IntelliJ IDEA Community/Ultimate
2. Plugins → Marketplace → ค้นหา "Cursive" → Install
3. Restart IDE
4. File → New Project → Clojure (Leiningen)
```

### Emacs + CIDER (ระดับ Pro)

```bash
# ติดตั้ง Emacs
brew install emacs  # macOS
sudo apt install emacs  # Ubuntu

# ใน .emacs หรือ init.el
(require 'package)
(add-to-list 'package-archives
             '("melpa" . "https://melpa.org/packages/"))
(package-initialize)

;; ติดตั้ง CIDER
M-x package-install RET cider RET
```

### Neovim + Conjure

```bash
# ใน init.lua หรือ init.vim
-- vim-plug
Plug 'Olical/conjure'
Plug 'guns/vim-sexp'
Plug 'tpope/vim-sexp-mappings-for-regular-people'
```

---

## ขั้นตอนที่ 5: Hello World - ครั้งแรก

### ทดสอบผ่าน REPL โดยตรง

```bash
# เปิด Clojure REPL
clojure

# หรือผ่าน Leiningen
lein repl
```

```clojure
;; ใน REPL พิมพ์:
(println "สวัสดี โลก!")
;; output: สวัสดี โลก!

;; ลองคำนวณ
(+ 1 2 3)
;; output: 6

(* 10 20)
;; output: 200

;; ออกจาก REPL
(exit)
;; หรือกด Ctrl+D
```

---

## ขั้นตอนที่ 6: สร้าง Project แรก

```bash
# สร้าง project ใหม่ด้วย Leiningen
lein new app hello-clojure

# โครงสร้างที่ได้:
# hello-clojure/
# ├── project.clj          # Project configuration
# ├── src/
# │   └── hello_clojure/
# │       └── core.clj     # Main source file
# ├── test/
# │   └── hello_clojure/
# │       └── core_test.clj # Test file
# └── README.md
```

```bash
# เข้าไปใน project
cd hello-clojure

# รันโปรแกรม
lein run
# output: Hello, World!
```

---

## ขั้นตอนที่ 7: ทำความเข้าใจ project.clj

```clojure
;; hello-clojure/project.clj
(defproject hello-clojure "0.1.0-SNAPSHOT"
  :description "My first Clojure project"
  :url "http://example.com/FIXME"
  :license {:name "EPL-2.0 OR GPL-2.0-or-later WITH Classpath-exception-2.0"
            :url "https://www.eclipse.org/legal/epl-2.0/"}
  :dependencies [[org.clojure/clojure "1.12.0"]]
  :main ^:skip-aot hello-clojure.core
  :target-path "target/%s"
  :profiles {:uberjar {:aot :all
                       :jvm-opts ["-Dclojure.compiler.direct-linking=true"]}})
```

**อธิบาย:**
- `defproject` - ประกาศ project
- `hello-clojure` - ชื่อ project
- `"0.1.0-SNAPSHOT"` - version
- `:dependencies` - dependencies ที่ต้องการ
- `:main` - namespace ที่มี `-main` function

---

## ขั้นตอนที่ 8: แก้ไข core.clj

```clojure
;; src/hello_clojure/core.clj

(ns hello-clojure.core)  ; ประกาศ namespace

(defn greet
  "ฟังก์ชันทักทาย"
  [name]
  (str "สวัสดี " name "! ยินดีต้อนรับสู่ Clojure!"))

(defn -main
  "Entry point ของโปรแกรม"
  [& args]
  (println (greet "โลก"))
  (println (greet "นักพัฒนา"))
  (println "Clojure version:" (clojure-version)))
```

```bash
# รัน
lein run
# output:
# สวัสดี โลก! ยินดีต้อนรับสู่ Clojure!
# สวัสดี นักพัฒนา! ยินดีต้อนรับสู่ Clojure!
# Clojure version: 1.12.0
```

---

## ขั้นตอนที่ 9: REPL-Driven Development

หัวใจสำคัญของการพัฒนาด้วย Clojure คือการใช้ REPL (Read-Eval-Print Loop)

```bash
# เปิด REPL ใน project
lein repl
```

```clojure
;; โหลด namespace ที่เราสร้าง
(require '[hello-clojure.core :as core])

;; เรียกใช้ฟังก์ชัน
(core/greet "Test")
;; output: "สวัสดี Test! ยินดีต้อนรับสู่ Clojure!"

;; แก้ไขโค้ดใน editor แล้ว reload
(require '[hello-clojure.core :as core] :reload)

;; ทดสอบทันที
(core/greet "World")
```

### REPL ทำงานอย่างไร?

```
User input  →  Reader  →  Evaluator  →  Printer  →  Loop
"(+ 1 2)"  →  (+ 1 2)  →    3       →   "3"     →  รอ input ใหม่
```

**Read**: แปลง text เป็น data structures
**Eval**: ประมวลผล data structures 
**Print**: แสดงผลลัพธ์
**Loop**: กลับไปรอ input ใหม่

---

## ขั้นตอนที่ 10: Syntax พื้นฐาน - S-Expressions

Clojure ใช้ S-Expressions (Symbolic Expressions) ซึ่งเป็น syntax ของ Lisp

```clojure
;; รูปแบบพื้นฐาน: (operator argument1 argument2 ...)
(+ 1 2)           ; => 3
(- 10 3)          ; => 7
(* 4 5)           ; => 20
(/ 10 2)          ; => 5

;; เรียกใช้ฟังก์ชัน
(println "Hello") ; เรียก println ด้วย argument "Hello"
(str "Hello" " " "World")  ; => "Hello World"

;; Nested expressions
(+ (* 2 3) (* 4 5))  ; => (6 + 20) = 26
(+ 1 (- 10 (/ 6 2))) ; => 1 + (10 - 3) = 8

;; ทุกอย่างคือ expression ที่มีค่า
(if true "yes" "no")  ; => "yes"
(if false "yes" "no") ; => "no"
```

### เปรียบเทียบกับภาษาอื่น

```javascript
// JavaScript
function add(a, b) { return a + b; }
add(1, 2)  // 3

// Python
def add(a, b): return a + b
add(1, 2)  # 3
```

```clojure
;; Clojure
(defn add [a b] (+ a b))
(add 1 2)  ; 3
```

---

## ขั้นตอนที่ 11: Comments

```clojure
;; นี่คือ comment บรรทัดเดียว (ใช้ ;; เป็น convention)
; comment แบบนี้ก็ได้ แต่ ;; นิยมกว่า

#_ (println "บรรทัดนี้จะถูก ignore ทั้งหมด")

#_(defn unused-function []
    "ฟังก์ชันนี้จะไม่ถูก compile")

;; Comment หลาย expression ด้วย (comment ...)
(comment
  (+ 1 2)
  (println "สามารถเขียน code ทดสอบใน comment block")
  ;; ใน VS Code + Calva: กด Ctrl+Enter จะ eval แต่ละ expression
  )
```

---

## ขั้นตอนที่ 12: Symbols และ Variables

```clojure
;; ประกาศ var (variable ระดับ namespace)
(def my-name "สมชาย")
(def age 25)
(def pi 3.14159)

;; ใช้งาน
(println my-name)  ; สมชาย
(println age)      ; 25
(println pi)       ; 3.14159

;; Clojure variables เป็น immutable โดย default
;; ไม่สามารถทำแบบนี้ได้:
; (set! my-name "สมหญิง")  ; ERROR!

;; ถ้าต้องการ mutable state ต้องใช้ atom
(def counter (atom 0))
(swap! counter inc)  ; เพิ่มค่าด้วย inc function
@counter  ; => 1 (อ่านค่าด้วย @)
```

---

## ขั้นตอนที่ 13: Naming Conventions ใน Clojure

```clojure
;; kebab-case สำหรับทุกอย่าง (ต่างจาก Java camelCase)
(def user-name "สมชาย")
(defn get-user-name [] user-name)
(defn calculate-total-price [items] ...)

;; คำถาม/predicates ใช้ ? ต่อท้าย
(defn empty? [coll] ...)
(defn valid-email? [email] ...)
(nil? nil)    ; => true
(empty? [])   ; => true

;; Destructive/mutation ใช้ ! ต่อท้าย (convention)
(swap! atom-var new-val)
(reset! atom-var 0)

;; Constants ใช้ ALL_CAPS (บางทีก็ไม่ได้ใช้ เพราะทุกอย่าง immutable)
(def MAX-CONNECTIONS 10)

;; Private ใช้ def- หรือ defn-
(defn- private-helper [] ...)

;; Namespace abbreviation
(require '[clojure.string :as str])
(str/upper-case "hello")  ; => "HELLO"
```

---

## ขั้นตอนที่ 14: Basic Arithmetic

```clojure
;; บวก ลบ คูณ หาร
(+ 1 2)        ; => 3
(- 10 3)       ; => 7
(* 4 5)        ; => 20
(/ 10 2)       ; => 5
(/ 7 2)        ; => 7/2 (Ratio! ไม่ใช่ 3.5)
(/ 7.0 2)      ; => 3.5 (ใช้ float เพื่อได้ decimal)

;; Modulo
(mod 10 3)     ; => 1
(rem 10 3)     ; => 1

;; Power
(Math/pow 2 10)  ; => 1024.0
(Math/sqrt 16)   ; => 4.0

;; Integer arithmetic
(quot 10 3)    ; => 3 (integer division)
(rem 10 3)     ; => 1 (remainder)

;; Multiple arguments
(+ 1 2 3 4 5)  ; => 15
(* 1 2 3 4 5)  ; => 120
(max 1 5 3 2)  ; => 5
(min 1 5 3 2)  ; => 1

;; Comparison
(= 1 1)        ; => true
(= 1 2)        ; => false
(not= 1 2)     ; => true
(< 1 2)        ; => true
(> 5 3)        ; => true
(<= 3 3)       ; => true
(>= 5 5)       ; => true

;; Chained comparison (Clojure รองรับ!)
(< 1 2 3 4)    ; => true (1 < 2 < 3 < 4)
(< 1 2 5 4)    ; => false (5 > 4)
```

---

## ขั้นตอนที่ 15: String Operations

```clojure
;; String literals
"Hello, World!"
"สวัสดี ชาวโลก"
"เส้นใหม่\nTab\t"

;; String functions (ต้อง require clojure.string)
(require '[clojure.string :as str])

;; ต่อ strings
(str "Hello" " " "World")           ; => "Hello World"
(str "หมายเลข: " 42)                ; => "หมายเลข: 42"

;; ความยาว
(count "Hello")                      ; => 5
(count "สวัสดี")                     ; => 6

;; ตัวพิมพ์ใหญ่/เล็ก
(str/upper-case "hello")             ; => "HELLO"
(str/lower-case "HELLO")             ; => "hello"

;; ตัดช่องว่าง
(str/trim "  hello  ")               ; => "hello"
(str/trim-newline "hello\n")         ; => "hello"

;; แยก/รวม
(str/split "a,b,c" #",")            ; => ["a" "b" "c"]
(str/join ", " ["a" "b" "c"])        ; => "a, b, c"

;; ค้นหา/แทนที่
(str/includes? "hello world" "world") ; => true
(str/starts-with? "hello" "hel")     ; => true
(str/ends-with? "hello" "llo")       ; => true
(str/replace "hello" "l" "r")        ; => "herro"
(str/replace-first "hello" "l" "r")  ; => "herlo"

;; แปลง
(str/reverse "hello")                ; => "olleh"

;; Format string
(format "ชื่อ: %s อายุ: %d" "สมชาย" 25)  ; => "ชื่อ: สมชาย อายุ: 25"
```

---

## ขั้นตอนที่ 16: Boolean และ Logical Operations

```clojure
;; Boolean values
true
false

;; Logical AND - ต้องเป็น truthy ทั้งหมด
(and true true)    ; => true
(and true false)   ; => false
(and false false)  ; => false

;; Logical OR - อย่างน้อยหนึ่งต้องเป็น truthy
(or true false)    ; => true
(or false false)   ; => false

;; Logical NOT
(not true)         ; => false
(not false)        ; => true

;; Truthy และ Falsy ใน Clojure
;; FALSY: เฉพาะ nil และ false เท่านั้น!
;; TRUTHY: ทุกอย่างที่ไม่ใช่ nil และ false

(if 0 "truthy" "falsy")      ; => "truthy" (0 เป็น truthy!)
(if "" "truthy" "falsy")     ; => "truthy" ("" เป็น truthy!)
(if [] "truthy" "falsy")     ; => "truthy" ([] เป็น truthy!)
(if nil "truthy" "falsy")    ; => "falsy"
(if false "truthy" "falsy")  ; => "falsy"

;; and/or คืนค่า (ไม่ใช่แค่ true/false)
(and 1 2 3)      ; => 3 (คืนค่าสุดท้ายที่ truthy)
(and 1 nil 3)    ; => nil (หยุดที่ falsy แรก)
(or nil false 3) ; => 3 (คืนค่าแรกที่ truthy)
(or nil false)   ; => false (คืนค่าสุดท้าย)
```

---

## ขั้นตอนที่ 17: nil และ null safety

```clojure
;; nil เทียบเท่า null ใน Java
nil                    ; => nil

;; ตรวจสอบ nil
(nil? nil)             ; => true
(nil? 0)               ; => false
(nil? "")              ; => false

;; some? - ตรงข้ามกับ nil?
(some? nil)            ; => false
(some? 0)              ; => true
(some? "hello")        ; => true

;; ป้องกัน NullPointerException
;; วิธีที่ 1: ใช้ when
(when (some? user)
  (println (:name user)))

;; วิธีที่ 2: ใช้ if-let
(if-let [name (:name user)]
  (println "ชื่อ:" name)
  (println "ไม่มีชื่อ"))

;; วิธีที่ 3: ใช้ some->
(some-> user :address :city str/upper-case)
;; ถ้า user หรือ :address หรือ :city เป็น nil จะคืน nil แทน error

;; วิธีที่ 4: ใช้ get ที่มี default value
(get user :name "ไม่ระบุชื่อ")  ; คืน "ไม่ระบุชื่อ" ถ้าไม่มี :name
```

---

## ขั้นตอนที่ 18: Clojure Data Types สรุป

```clojure
;; Integers (Long)
42
-10
1000000N    ; BigInteger (N suffix)

;; Floating point (Double)
3.14
-2.5
1.5e10      ; Scientific notation

;; Ratio (ไม่ truncate!)
1/3         ; => 1/3 (exact!)
22/7        ; => 22/7

;; String
"Hello"
"Thai: สวัสดี"

;; Character
\a          ; => \a
\space      ; => \space
\newline    ; => \newline

;; Boolean
true
false

;; Nil
nil

;; Symbol (ชื่อ)
foo
my-symbol

;; Keyword (immutable identifiers นำหน้าด้วย :)
:name
:age
:user/name  ; Qualified keyword ด้วย namespace

;; Collections
[1 2 3]           ; Vector
(1 2 3)           ; List (in code)
'(1 2 3)          ; Quoted list (data)
{:a 1 :b 2}       ; Map
#{1 2 3}          ; Set
```

---

## ขั้นตอนที่ 19: ทำความเข้าใจ Immutability

หนึ่งในแนวคิดสำคัญที่สุดของ Clojure คือ **Immutability**

```clojure
;; ใน Clojure ข้อมูลไม่เปลี่ยนแปลง
(def numbers [1 2 3 4 5])

;; conj ไม่ได้แก้ไข numbers
;; แต่สร้าง vector ใหม่
(def more-numbers (conj numbers 6))

numbers        ; => [1 2 3 4 5] (ไม่เปลี่ยน!)
more-numbers   ; => [1 2 3 4 5 6]

;; เปรียบเทียบกับ Java
;; Java: list.add(6); // list ถูกเปลี่ยน!
;; Clojure: (conj list 6) ; สร้างใหม่ list เดิมไม่เปลี่ยน

;; ประโยชน์:
;; 1. Thread-safe โดย default
;; 2. ง่ายต่อการ debug (state ไม่เปลี่ยน)
;; 3. Undo/redo ทำได้ง่าย
;; 4. Cacheable
```

---

## ขั้นตอนที่ 20: First Class Functions

ใน Clojure ฟังก์ชันเป็น "First Class Citizens" หมายถึงสามารถส่งเป็น argument ได้

```clojure
;; ฟังก์ชันคือข้อมูล - สามารถเก็บใน var
(def double (fn [x] (* 2 x)))
(double 5)  ; => 10

;; ใช้ defn (syntactic sugar สำหรับ def + fn)
(defn double [x] (* 2 x))
(double 5)  ; => 10

;; ส่งฟังก์ชันเป็น argument
(map double [1 2 3 4 5])
; => (2 4 6 8 10)

;; ฟังก์ชันคืนค่าฟังก์ชัน (Higher-order functions)
(defn make-multiplier [n]
  (fn [x] (* n x)))

(def triple (make-multiplier 3))
(triple 5)  ; => 15
(triple 10) ; => 30
```

---

## ขั้นตอนที่ 21: เริ่มใช้ VS Code + Calva

```
1. เปิด VS Code
2. เปิด folder ของ project
3. กด Ctrl+Shift+P (Command Palette)
4. พิมพ์ "Calva: Start a Project REPL and Connect"
5. เลือก "Leiningen"
6. รอ REPL เริ่มต้น
```

### Keyboard Shortcuts ที่สำคัญ (Calva)

```
Ctrl+Enter          = Evaluate current form
Alt+Enter           = Evaluate top-level form  
Ctrl+Alt+C, C       = Evaluate entire namespace
Ctrl+Alt+C, E       = Evaluate to comment

Ctrl+Alt+C, Space   = Format code
Alt+Left/Right      = Navigate by sexp

;; ใน REPL
Up/Down Arrow       = History navigation
Escape              = Clear input
```

---

## ขั้นตอนที่ 22: deps.edn (Alternative Build Tool)

นอกจาก Leiningen ยังมี `deps.edn` ซึ่งเป็น official Clojure CLI

```bash
# สร้าง project ด้วย deps.edn
mkdir my-project
cd my-project
```

```clojure
;; deps.edn
{:deps {org.clojure/clojure {:mvn/version "1.12.0"}}
 :paths ["src"]
 :aliases
 {:dev {:extra-paths ["dev"]
        :extra-deps {nrepl/nrepl {:mvn/version "1.3.0"}
                     cider/cider-nrepl {:mvn/version "0.50.2"}}}
  :test {:extra-paths ["test"]
         :extra-deps {io.github.cognitect-labs/test-runner
                      {:git/url "https://github.com/cognitect-labs/test-runner.git"
                       :sha "9e25b8e8fb659e9d5d96e56e0b95b45c98ce8e33"}}}}}
```

```bash
# รัน REPL
clj -M:dev

# รัน tests
clj -M:test -m cognitect.test-runner

# รัน specific namespace
clj -M -m my.main
```

---

## ขั้นตอนที่ 23: แนวคิด Pure Functions

```clojure
;; Pure Function: ผลลัพธ์ขึ้นอยู่กับ arguments เท่านั้น
;; ไม่มี side effects

;; ✅ Pure function
(defn add [a b]
  (+ a b))

(add 3 4)  ; => 7 เสมอ ไม่ว่าจะเรียกกี่ครั้ง

;; ❌ Impure function (มี side effect)
(def counter (atom 0))

(defn add-and-count [a b]
  (swap! counter inc)  ; side effect!
  (+ a b))

;; ❌ Impure function (ขึ้นอยู่กับ external state)
(defn get-greeting []
  (str "Hello, " @current-user))  ; ขึ้นอยู่กับ current-user

;; ✅ ทำให้ pure โดย inject dependencies
(defn get-greeting [user]
  (str "Hello, " user))

;; ประโยชน์ของ Pure Functions:
;; 1. ทดสอบง่าย - รู้ output จาก input
;; 2. Debug ง่าย - ไม่มี hidden state
;; 3. Cacheable (memoization)
;; 4. Thread-safe
;; 5. Composable
```

---

## ขั้นตอนที่ 24: ทดสอบโค้ดเบื้องต้น

```clojure
;; test/hello_clojure/core_test.clj

(ns hello-clojure.core-test
  (:require [clojure.test :refer :all]
            [hello-clojure.core :refer :all]))

;; เขียน test
(deftest test-greet
  (testing "greet function"
    (is (= "สวัสดี โลก! ยินดีต้อนรับสู่ Clojure!"
           (greet "โลก")))
    (is (= "สวัสดี สมชาย! ยินดีต้อนรับสู่ Clojure!"
           (greet "สมชาย")))))

(deftest test-add
  (testing "basic addition"
    (is (= 3 (+ 1 2)))
    (is (= 0 (+ -1 1)))
    (is (pos? (+ 1 2)))))
```

```bash
# รัน tests
lein test

# รัน test เฉพาะ namespace
lein test hello-clojure.core-test
```

---

## ขั้นตอนที่ 25: Build Executable JAR

```bash
# Build uberjar (executable JAR ที่มี dependencies ครบ)
lein uberjar

# รัน JAR
java -jar target/uberjar/hello-clojure-0.1.0-SNAPSHOT-standalone.jar

# output:
# สวัสดี โลก! ยินดีต้อนรับสู่ Clojure!
# สวัสดี นักพัฒนา! ยินดีต้อนรับสู่ Clojure!
```

---

## ขั้นตอนที่ 26: นำข้อมูลใส่โปรแกรมจาก Command Line

```clojure
;; src/hello_clojure/core.clj
(ns hello-clojure.core)

(defn greet [name]
  (str "สวัสดี " name "! ยินดีต้อนรับสู่ Clojure!"))

(defn -main [& args]
  (if (seq args)
    (doseq [name args]
      (println (greet name)))
    (println (greet "โลก"))))
```

```bash
# รันพร้อม arguments
lein run สมชาย สมหญิง

# output:
# สวัสดี สมชาย! ยินดีต้อนรับสู่ Clojure!
# สวัสดี สมหญิง! ยินดีต้อนรับสู่ Clojure!
```

---

## ขั้นตอนที่ 27: การ Debug เบื้องต้น

```clojure
;; ใช้ println ง่ายที่สุด
(defn calculate [x y]
  (println "x =" x "y =" y)  ; debug print
  (let [result (* x y)]
    (println "result =" result)  ; debug print
    result))

;; ใช้ tap> และ add-tap (Clojure 1.10+)
(add-tap (fn [v] (println "TAP:" v)))
(tap> {:user "สมชาย" :age 25})
;; output: TAP: {:user "สมชาย", :age 25}

;; ใช้ clojure.tools.trace
;; เพิ่ม dependency ก่อน:
;; [org.clojure/tools.trace "0.7.11"]

(require '[clojure.tools.trace :refer [trace]])

(defn fibonacci [n]
  (trace n)  ; แสดงทุก call
  (if (<= n 1)
    n
    (+ (fibonacci (- n 1))
       (fibonacci (- n 2)))))
```

---

## ขั้นตอนที่ 28: Clojure Cheatsheet สรุป Part 1

```clojure
;; === BASIC TYPES ===
42          ; Integer (Long)
3.14        ; Double
1/3         ; Ratio
"hello"     ; String
\a          ; Character
:keyword    ; Keyword
true false  ; Boolean
nil         ; Null

;; === ARITHMETIC ===
(+ 1 2)     ; 3
(- 5 3)     ; 2
(* 4 5)     ; 20
(/ 10 2)    ; 5
(mod 7 3)   ; 1
(inc 5)     ; 6  (increment)
(dec 5)     ; 4  (decrement)
(abs -5)    ; 5

;; === STRINGS ===
(str "a" "b")           ; "ab"
(count "hello")         ; 5
(subs "hello" 1 3)     ; "el"

;; === COMPARISONS ===
(= 1 1)     ; true
(not= 1 2)  ; true
(< 1 2)     ; true
(> 5 3)     ; true

;; === LOGIC ===
(and true false)   ; false
(or true false)    ; true
(not true)         ; false

;; === DEFINITIONS ===
(def x 42)                      ; var
(defn add [a b] (+ a b))       ; function
(fn [x] (* x 2))               ; anonymous function

;; === CONTROL FLOW ===
(if condition true-val false-val)
(when condition body)
(cond
  test1 val1
  test2 val2
  :else default)
```

---

## ขั้นตอนที่ 29: Project Exercise - Calculator

สร้าง Calculator อย่างง่ายเป็น exercise แรก

```clojure
;; src/calculator/core.clj

(ns calculator.core)

(defn calculate
  "ฟังก์ชันคำนวณพื้นฐาน"
  [op a b]
  (case op
    :add      (+ a b)
    :subtract (- a b)
    :multiply (* a b)
    :divide   (if (zero? b)
                (throw (ArithmeticException. "ไม่สามารถหารด้วยศูนย์"))
                (/ a b))
    (throw (IllegalArgumentException. (str "ไม่รู้จัก operation: " op)))))

(defn format-result
  "แสดงผลลัพธ์ในรูปแบบที่อ่านง่าย"
  [op a b result]
  (let [op-symbol (case op
                    :add "+"
                    :subtract "-"
                    :multiply "×"
                    :divide "÷")]
    (format "%s %s %s = %s" a op-symbol b result)))

(defn -main [& args]
  (println "=== Clojure Calculator ===")
  
  ;; ทดสอบการคำนวณ
  (doseq [[op a b] [[:add 10 5]
                     [:subtract 10 3]
                     [:multiply 4 7]
                     [:divide 20 4]]]
    (let [result (calculate op a b)]
      (println (format-result op a b result))))
  
  ;; ทดสอบ error handling
  (try
    (calculate :divide 10 0)
    (catch ArithmeticException e
      (println "Error:" (.getMessage e)))))
```

```bash
# รัน
lein run
# output:
# === Clojure Calculator ===
# 10 + 5 = 15
# 10 - 3 = 7
# 4 × 7 = 28
# 20 ÷ 4 = 5
# Error: ไม่สามารถหารด้วยศูนย์
```

---

## ขั้นตอนที่ 30: Summary และ Next Steps

### สิ่งที่เรียนรู้ใน Part 1:
1. ✅ ติดตั้ง Java, Clojure, Leiningen
2. ✅ สร้าง project แรก
3. ✅ เข้าใจ REPL-driven development
4. ✅ S-Expressions syntax
5. ✅ Basic types: numbers, strings, booleans, nil
6. ✅ Basic operations: arithmetic, string, logical
7. ✅ Immutability concept
8. ✅ Pure functions concept
9. ✅ First class functions concept
10. ✅ สร้าง executable JAR

### Key Principles ที่ต้องจำ:
```
1. Everything is an Expression - ทุกอย่างมีค่า
2. Immutability by Default - ข้อมูลไม่เปลี่ยน
3. Functions are First Class - ฟังก์ชันคือข้อมูล
4. REPL-Driven Development - interactive feedback
5. Simplicity > Cleverness - เรียบง่ายกว่าฉลาด
```

### Exercises:

**Exercise 1**: สร้างฟังก์ชัน `temperature-converter` ที่แปลงอุณหภูมิ
```clojure
(temperature-converter :celsius->fahrenheit 100)  ; => 212.0
(temperature-converter :fahrenheit->celsius 212)  ; => 100.0
```

**Exercise 2**: สร้างฟังก์ชัน `bmi-calculator` ที่คำนวณ BMI
```clojure
(bmi-calculator 70 1.75)  ; => 22.857... 
;; พร้อมบอก category: Underweight/Normal/Overweight/Obese
```

**Exercise 3**: สร้าง FizzBuzz
```clojure
;; print 1 ถึง 100
;; ถ้าหาร 3 ลงตัว print "Fizz"
;; ถ้าหาร 5 ลงตัว print "Buzz"
;; ถ้าหาร 15 ลงตัว print "FizzBuzz"
;; อื่นๆ print ตัวเลข
```

---

### อ่านต่อใน Part 2: REPL และ Interactive Development →

---

*Part 1 จาก 100+ | ขั้นตอน 1-30 จาก 1000+*
