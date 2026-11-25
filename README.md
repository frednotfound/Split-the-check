# 分帳小幫手

多人旅遊與聚餐的分帳網頁工具。整個網站只有一個 `index.html`，放到 GitHub Pages 就能使用。

- **不設定任何東西**：本地帳本、匯出／匯入備份、「網址即存檔」分享連結、文字報告與結算圖片全部可用，完全不需要伺服器或登入。
- **加上 Firebase（選用，免費方案即可）**：多一個「多人即時房間」。朋友點邀請連結、輸入自己的名字，就能各自用手機記帳，所有人的畫面即時同步。

---

## 一、部署到 GitHub Pages

1. 在 GitHub 建立一個新的 repository（例如 `split-bill`），設為 Public。
2. 上傳 `index.html`（README 可一起上傳）。
3. 到 **Settings → Pages**，Source 選 **Deploy from a branch**，Branch 選 `main`，資料夾選 `/ (root)`，按 Save。
4. 等一兩分鐘，網址會是 `https://你的帳號.github.io/split-bill/`。

只要本地帳本和分享連結的話，做到這裡就完成了。

---

## 二、（選用）開啟多人即時房間

### 1. 建立 Firebase 專案與資料庫

1. 到 [Firebase Console](https://console.firebase.google.com/) → **新增專案**（Google Analytics 可以關閉）。
2. 左側選單 **建構 → Firestore Database → 建立資料庫**。
   - 位置建議選 `asia-east1`（台灣），建立後無法更改。
   - 模式選 **正式版模式（production mode）**。

### 2. 貼上安全規則

在 Firestore 的 **規則（Rules）** 分頁，整段換成下面內容後按 **發布**：

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function validRoom(id) { return id.matches('^[a-z0-9]{12,40}$'); }
    function roomExists(id) { return existsAfter(/databases/$(database)/documents/rooms/$(id)); }

    match /rooms/{roomId} {
      // 知道房間 ID 才能讀；禁止列出所有房間、禁止刪除房間
      allow get: if validRoom(roomId);
      allow list, delete: if false;
      allow create, update: if validRoom(roomId)
        && request.resource.data.keys().hasOnly(['name','mainCurrency','currencies','rounding','settled','createdAt','updatedAt','v'])
        && request.resource.data.name is string
        && request.resource.data.name.size() <= 60;

      match /members/{memberId} {
        allow read: if validRoom(roomId);
        allow delete: if validRoom(roomId);
        allow create, update: if validRoom(roomId) && roomExists(roomId)
          && request.resource.data.keys().hasOnly(['name','createdAt'])
          && request.resource.data.name is string
          && request.resource.data.name.size() <= 30;
      }

      match /expenses/{expenseId} {
        allow read: if validRoom(roomId);
        allow delete: if validRoom(roomId);
        allow create, update: if validRoom(roomId) && roomExists(roomId)
          && request.resource.data.keys().hasOnly(['title','amount','currency','payer','date','mode','participants','shares','items','service','discount','note','createdAt','updatedAt'])
          && request.resource.data.title is string
          && request.resource.data.title.size() <= 100
          && request.resource.data.amount is int
          && request.resource.data.amount >= 0;
      }
    }
  }
}
```

### 3. 取得網頁設定並貼進 index.html

1. Firebase Console → 左上齒輪 **專案設定 → 一般**，往下找到「你的應用程式」，按網頁圖示 **`</>`**，隨便取個暱稱後註冊（不需要勾選 Hosting）。
2. 畫面會顯示一段 `const firebaseConfig = { ... }`。
3. 打開 `index.html`，搜尋 `const FIREBASE_CONFIG = null;`，把它換成你的設定，例如：

```js
const FIREBASE_CONFIG = {
  apiKey: "AIza....",
  authDomain: "your-project.firebaseapp.com",
  projectId: "your-project",
  storageBucket: "your-project.appspot.com",
  messagingSenderId: "1234567890",
  appId: "1:1234567890:web:abcdef"
};
```

4. 重新上傳 `index.html` 到 GitHub。打開網站，「結算與分享」最下方就會出現「建立房間並取得邀請連結」。

> `apiKey` 放在網頁裡是正常的，它只用來辨識專案，不是密碼。真正保護資料的是上面的安全規則。
> 想再更保險，可以到 Google Cloud Console → API 與服務 → 憑證，把這組 API 金鑰限制為只能從 `https://你的帳號.github.io/*` 使用。

---

## 三、使用方式

| 想做的事 | 在哪裡 |
|---|---|
| 新增成員、標記「我」 | 成員與設定 → 成員 |
| 設定外幣匯率 | 成員與設定 → 幣別與匯率 |
| 記一筆（均分／份數／各自點單＋服務費／折扣） | 右下角「記一筆」 |
| 看誰該轉多少給誰、標記已結清 | 結算與分享 → 轉帳建議 |
| 複製 LINE 文字、產生結算圖片 | 結算與分享 → 分享結算 |
| 把整本帳本傳給朋友看（不需伺服器） | 結算與分享 → 分享帳本連結 |
| 大家一起即時記帳 | 結算與分享 → 多人即時房間（需設定 Firebase） |
| 換手機前備份 | 成員與設定 → 匯出全部帳本備份 |

### 兩種分享方式的差別

- **分享帳本連結**：把整本帳本壓縮進網址，傳出去的是「當下的快照」。朋友可以看，也可以存成自己的帳本後繼續編輯，但之後雙方的修改不會互相同步。適合「一個人記帳，結束後丟給大家看」。
- **多人即時房間**：資料存在 Firebase，每個人都能新增、修改，所有人的畫面即時更新。適合「旅途中大家各自記帳」。

### 計算方式

- 所有金額都以「分」為單位、用整數計算，外幣換算與分攤都不會有浮點誤差。
- 分攤除不盡時，使用最大餘數法把零頭分給餘數最大的人，每筆消費的分攤加總一定等於總額。
- 選擇「整數」顯示時，四捨五入後如果出現 1 元左右的尾差，會自動分配，保證所有人應收、應付的總和剛好為 0。
- 轉帳建議會先抵銷交叉墊付，再找出轉帳次數最少的方案。還有餘額沒結清的成員在 16 人以內時，保證是最少次數；超過時會改用快速的近似演算法，最多需要「人數 − 1」筆轉帳。

### 注意事項

- 資料存在瀏覽器的 localStorage。清除瀏覽資料、使用無痕模式，或在 LINE 內建瀏覽器和 Safari 之間切換時，看到的會是不同的資料。重要帳本請記得匯出備份。
- 「我的收款資訊」只存在你自己的裝置，不會放進分享連結，也不會上傳到房間。
- 拿到房間邀請連結的人都能編輯帳本，請只分享到一起分帳的群組。
