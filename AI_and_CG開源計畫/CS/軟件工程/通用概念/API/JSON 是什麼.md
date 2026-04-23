JSON（JavaScript Object Notation）是一種**輕量級的純文字資料交換格式**， JavaScript 物件的標準格式，常用於網站上的資料呈現、傳輸 (例如將資料從伺服器送至用戶端，以利顯示網頁)

它的設計目標是：

- 結構清楚
- 易於閱讀
- 容易被程式解析

「JSON 物件基本上就是 JavaScript 物件」，而這敘述在大多數情況下都對。如同標準的 JavaScript 物件，當然可在 JSON 之內加入相同的基本資料類型，如字串、數字、陣列、布林值，以及其他物件，接著同樣能再建構出資料繼承，如：

```json
{
  "squadName": "Super hero squad",
  "homeTown": "Metro City",
  "formed": 2016,
  "secretBase": "Super tower",
  "active": true,
  "members": [
    {
      "name": "Molecule Man",
      "age": 29,
      "secretIdentity": "Dan Jukes",
      "powers": ["Radiation resistance", "Turning tiny", "Radiation blast"]
    },
    {
      "name": "Madame Uppercut",
      "age": 39,
      "secretIdentity": "Jane Wilson",
      "powers": [
        "Million tonne punch",
        "Damage resistance",
        "Superhuman reflexes"
      ]
    },
    {
      "name": "Eternal Flame",
      "age": 1000000,
      "secretIdentity": "Unknown",
      "powers": [
        "Immortality",
        "Heat Immunity",
        "Inferno",
        "Teleportation",
        "Interdimensional travel"
      ]
    }
  ]
}
```

----

### XML VS JSON
 
| 格式   | 可讀性 | 資料大小 | 解析速度 | 使用場景         |
| ---- | --- | ---- | ---- | ------------ |
| JSON | 高   | 小    | 快    | REST API（主流） |
| XML  | 中   | 大    | 慢    | SOAP、金融、舊系統  |

JSON 資料相比 XML 體積更小傳輸更快，且更容易閱讀

推薦參考資料：
* [說真的，到底什麼是 JSON？](https://developer.mozilla.org/zh-TW/docs/Learn_web_development/Core/Scripting/JSON)
