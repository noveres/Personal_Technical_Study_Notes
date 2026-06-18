想像一下日常生活中「開」這個字：
- 開 (大門) $\rightarrow$ 用手推或用鑰匙解鎖。
- 開 (電視) $\rightarrow$ 按下遙控器的電源鈕。
- 開 (車子) $\rightarrow$ 踩油門、轉動方向盤。
- 開 (支票) $\rightarrow$ 拿筆簽名寫上金額。

雖然動作的名字都叫「開」**，但因為後面接的**「對象（參數）」**不同，實際上執行的**「具體步驟（內部邏輯）」就完全不同。這就是重載！

## C++ 範例

如果沒有重載，當想寫一個列印函數時，必須為不同的資料型態取不同的名字：

```C++
void printInt(int i);
void printString(string s);
void printDouble(double d);
```

這樣呼叫時非常麻煩，你必須自己記住哪種資料要配哪個函數。
**有了「重載」，只需要定義一個名字：**

```C++
#include <iostream>
#include <string>


// 1. 接受整數的 print
void print(int i) {
    std::cout << "這是整數: " << i << std::endl;
}

// 2. 接受字串的 print (名稱相同，參數型態不同)
void print(string s) {
    std::cout << "這是字串: " << s << std::endl;
}

// 3. 接受兩個整數的 print (名稱相同，參數數量不同)
void print(int a, int b) {
    std::cout << "這是兩個整數: " << a << " 和 " << b << std::endl;
}

int main() {
    print(100);       // 編譯器自動調用第 1 個
    print("Hello");   // 編譯器自動調用第 2 個
    print(10, 20);    // 編譯器自動調用第 3 個
    return 0;
}
```

## 重載的硬性規則

透過 **函數簽名（Function Signature）** 來判定的。要構成重載，函數的名字必須相同，但**參數列表必須有明確的分別**。

必須符合以下至少一個條件：

1. **參數的「型態」不同**（例如：`int` vs `string`）。
2. **參數的「數量」不同**（例如：1 個參數 vs 2 個參數）。
3. **參數的「順序」不同**（例如：`print(int, double)` vs `print(double, int)`）。
    

###  絕對不能作為重載依據的雷區：

- **只有「回傳值型態（Return Type）」不同，不能構成重載！**

    ```C++
    int getValue();
    string getValue(); 
    // 編譯器會報錯，因為當你直接呼叫 getValue(); 時，系統無法判斷想拿哪種回傳值。
    ```

##  重載的好處

- **提高可讀性：** 概念相同的行為，不需要硬生出各種古怪的函數名稱（如 `addInt`, `addFloat`, `addVector`），通通叫 `add` 即可。
- **降低記憶成本：** 開發者（或調用你程式碼的人）只需要記住一個核心函數名稱，就能處理多種不同的資料情境。
- **與構造函數（Constructor）結合：** 這也是為什麼類別可以有多個構造函數
	例如：預設構造函數、帶參數的構造函數，讓我們能用各種不同的方式初始化一個物件。