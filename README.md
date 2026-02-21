# OurScheme（Scheme Interpreter in C++）

這個專案是一個用 C++ 實作的 **簡化版 Scheme 直譯器 / REPL**（互動式讀取—解析—求值—輸出）。  
程式會讀入 S-expression，做詞彙切分與語法檢查，建立樹狀資料結構，並支援一組內建 procedure 與錯誤回報。


---

## 功能概覽

### 1) REPL 流程
每次輸入一個 S-expression：
1. **Tokenize**：`Get_token()` / `Check()` 將輸入拆成 token（含 line/column 記錄）
2. **Grammar 檢查**：`Grammer()` 檢查括號、dot、quote 等語法正確性
3. **建樹**：`BuildDS()` 依據 token 串建立 `DATA` 樹（down 表子節點、next 表同層下一個）
4. **求值**：`Semantics()` / `Evaluate()` 執行內建 procedure、處理 define / lambda / let / if / cond 等
5. **輸出**：`Print()` / `PrintDS()` 以格式化方式列印結果或錯誤

輸入 `(exit)` 或到達 EOF 會結束。

---

## 內建資料型別（token）
程式以整數常數表示 token 類型（節錄）：

- `LEFT` / `RIGHT`：`(` / `)`
- `INT` / `FLOAT`
- `STRING`
- `SYMBOL`
- `DOT`：`.`
- `NIL`：`nil` / `()` / `#f`
- `T`：`#t`
- `QUOTE`：`'` / `quote`

---

## 支援的內建 procedures

程式在 `Load()` 內載入一組預設指令（總數 `cmdNum = 41`），常見如下：

### List / Quote
- `cons`, `list`, `quote`, `car`, `cdr`

### Arithmetic
- `+`, `-`, `*`, `/`

### Logic
- `not`, `and`, `or`

### Compare
- `>`, `<`, `>=`, `<=`, `=`
- `string-append`, `string>?`, `string<?`, `string=?`

### Equality
- `eqv?`, `equal?`

### Control / Special Forms
- `begin`, `if`, `cond`, `let`, `lambda`
- `define`
- `clean-environment`
- `exit`

> 註：程式內部把 procedure 印成 `#<procedure xxx>` 的形式，並用 `mean/inter` 等旗標區分內建與一般節點。

---

## 範例用法

### 基本運算
```scheme
(+ 1 2 3)
(* 2 3)
(/ 10 2)
```

### List 操作
```scheme
(cons 1 (list 2 3))
(car (list 10 20 30))
(cdr (list 10 20 30))
```

### 條件判斷
```scheme
(if #t 1 2)
(cond ((> 3 5) 100)
      (else 200))
```

### define（定義變數 / 函數）
```scheme
(define x 10)
(+ x 5)

(define (add2 a b) (+ a b))
(add2 3 4)
```

### lambda / let
```scheme
((lambda (x) (+ x 1)) 10)

(let ((a 1) (b 2))
  (+ a b))
```

### 結束
```scheme
(exit)
```

---

## 輸入格式注意事項

- 支援 `;` 行內註解（`;` 後面會被忽略）
- `"` 字串需成對；若缺少 closing quote 會報錯
- `DoNum()` 會把浮點數格式化到小數點後 3 位
- `'` 會在 `Simplify()` 被改寫為 `(quote ...)` 的結構

---
## 錯誤處理（Error messages）
主要錯誤由 Error(int &e) 統一輸出，常見包含：
- `ERROR (no closing quote)`
- `ERROR (no more input)（EOF）`
- `ERROR (unexpected token)`（期待 atom 或 (、或期待 )）
- `ERROR (unbound symbol)`
- `ERROR (non-list)`
- `ERROR (attempt to apply non-function)`
- `ERROR (DEFINE/LAMBDA/LET/COND format)`
- `ERROR (incorrect number of arguments)`
- `ERROR (xxx with incorrect argument type)`
- `ERROR (no return value)`
- `ERROR (division by zero)`

錯誤通常會帶上 token 的 Line/Column 或印出對應的 S-expression 片段。

---

## 專案結構（核心設計）
### `PL::DATA` 節點

每個節點代表一個 token 或子樹：
- `token`：型別
- `element[256]`：文字內容（例如 +、"abc"、(）
- `line`, `column`：來源位置
- `next`：同層下一個（類似 linked list）
- `down`：子層（list 內容）
- `addr`：用於 eqv?、lambda 綁定等用途的識別

### 重要成員指標（用途簡述）

- `m_current`：token 串 / 或建樹中游標
- `m_head`：當前 expression 的樹根
- `m_function`：全域 binding（define 的符號表 + 內建 procedure）
- `m_para`：已定義的 function（含 lambda 的定義資訊）
- `m_tmpara`：let/lambda 呼叫時的參數綁定鏈
- `m_error`：錯誤定位用節點
- `m_cond`：cond/if/let 求值時暫存結果


## 編譯與執行
### 編譯（g++）
```scheme
g++ -std=c++11 -O2 -o ourscheme main.cpp
```
### 執行
```scheme
./ourscheme
```

程式從 stdin 讀入測資/互動輸入。
注意：程式一開始會 cin >> gTestNum; 讀入一個整數（測資編號/測試次數標記），因此你的輸入第一個 token 需要是整數，例如：
```scheme
1
(+ 1 2)
(exit)
```
