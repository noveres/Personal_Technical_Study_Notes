，**靜態類別**就是一種**無法實例化**的特殊類別。

### 1. 什麼是「無法實例化」 (Cannot be Instantiated)?

在 OOP 中，「實例化（Instantiate）」就是使用 `new` 關鍵字把（Class）變成一個真實存在的「物件（Object/Instance）」。

- **實例化**: `const myCar = new Car();` (你根據車的設計圖，造出了一台真正的車)。
    
- **無法實例化**: 編譯器直接禁止你寫 `new ClassName()`。如果你嘗試這樣做，程式會報錯。
    

#### 為什麼要禁止實例化？

通常有兩種情況我們不需要（或不應該）產生「物件」：

1. **純工具庫 (Static Class)**：裡面全都是功能函數（如數學計算），不需要記住狀態，不需要產生多個副本。
    
2. **抽象概念 (Abstract Class)**：這是一個未完成的基底，必須被繼承並實作細節後，才能透過子類別來實例化（例如：「動物」這個概念無法實例化，但「狗」可以）。
    

---

### 2. 什麼是「靜態類別」 (Static Class)?

靜態類別是一種**只能包含靜態成員**（Static Members）的類別。它存在的唯一目的就是**作為一個容器**，把相關的工具函數或常數放在一起。

#### 核心特徵：

1. **不能使用 `new`**：正如上述，你不能創造它的實體。
    
2. **直接呼叫**：你不需要建立物件，直接用 `類別名稱.方法()` 就可以使用。
    
3. **全域共用**：它在記憶體中只有一份，所有人都共用同一個。
    

#### 最經典的例子：`Math`

在 JavaScript/TypeScript 或 C# 中，`Math` 就是最典型的靜態類別概念（雖然 JS 中它是個內建物件，但行為邏輯一樣）。

- ❌ **錯誤用法（試圖實例化）**：
    
    你從來不需要造一個「數學物件」來算圓周率。
    
    ``` TypeScript
    // Error: Math is not a constructor
    const myMath = new Math(); 
    ```
    
- ✅ **正確用法（直接使用）**：
    
    直接呼叫它提供的工具。
  ``` TypeScript
   const result = Math.abs(-5); // 直接用 Class Name 呼叫
   const pi = Math.PI;
  ```
    

---

### 3. 程式碼比較 (TypeScript/C# 概念)

讓我們用一個「寫 Log 工具」的例子來對比：

#### 一般類別 (Normal Class) - 需要實例化

這就像是你買了一本筆記本，你必須先買（new）這本筆記本，才能在上面寫字。每個人買的筆記本是獨立的。

``` TypeScript
class Logger {
    log(message: string) {
        console.log(message);
    }
}

// 必須先 new 出來才能用
const myLogger = new Logger(); 
myLogger.log("Hello"); 
```

#### 靜態類別 (Static Class) - 無法實例化

這就像是一個掛在牆上的公共廣播系統。你不需要「買」一個廣播系統，你走過去直接按按鈕說話，大家都能聽到。

``` TypeScript
// 在 C# 中會寫 static class Logger
// 在 TS 中通常用 static 方法或是直接 export const function
class StaticLogger {
    // 私有建構函式，強制防止外部 new 它
    private constructor() {} 

    static log(message: string) {
        console.log("System Alert: " + message);
    }
}

// ❌ 報錯！無法實例化
// const log = new StaticLogger(); 

// ✅ 直接使用
StaticLogger.log("Server Down!");
```

### 總結

|**特性**|**一般類別 (Instance Class)**|**靜態類別 (Static Class)**|
|---|---|---|
|**能用 `new` 嗎？**|可以|**不行 (無法實例化)**|
|**資料狀態**|每個物件有自己獨立的資料 (Stateful)|全程式共用一份資料 (Stateless/Shared)|
|**使用方式**|`instance.method()`|`ClassName.method()`|
|**生活譬喻**|**筆記本** (你要先買一本才能寫，你的筆記本跟我的不同)|**維基百科** (你不需要下載整個網站，大家上網查看到的都是同一份)|

簡單來說，如果你發現某個功能**不需要記住任何資料，只是單純的「輸入 A 得到 B」**（例如日期格式轉換、字串處理），那它就適合寫成靜態類別（或單純的 Export Function），這時它就具備「無法實例化」的特性。