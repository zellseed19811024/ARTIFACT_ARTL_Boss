# Artale Boss Timer

楓之谷 Artale 打王重生計時小工具（Vue 3 + Vite + Tailwind）。

## 操作說明

| 項目 | 說明 |
|------|------|
| **新增記錄** | 右下角選 BOSS → 按 `+` → 填 **頻道**、**擊殺時間**（24 小時制）→ 新增。同一 BOSS + 同一頻道會覆寫舊紀錄。 |
| **列表分區** | **即將重生**：未到最早重生時間；**可能已重生**：在重生時間區間內；**已重生**：已過最晚重生時間。 |
| **已重生區** | 預設收合，按 **展開** 才顯示完整列表；展開後可按 **收合** 隱藏。 |
| **排序** | 各區標題旁可選 **時間**（依最早重生時間）或 **頻道**（數字順序）。 |
| **BOSS 最愛** | 下拉選單按星星收藏（可多個）；收藏會自動置頂。 |
| **霓虹綠列** | 僅在 **頻道** 排序時生效：與上一筆或下一筆頻道號差距 ≤ 頁尾設定的 **N**，該列會標示。 |
| **眼睛** | 「可能已重生」區：點擊記錄「已去看過、沒看到王」；點時間戳可刪除該次踩點。 |
| **勾選圖示** | 以 **現在時間** 當新擊殺，重算重生（等同剛打完）。 |
| **垃圾桶** | 刪除該筆記錄。 |
| **分享／匯入** | 複製全部紀錄給他人；貼上他人資料會與本地 **合併**（同 BOSS+頻道保留較新擊殺時間）。 |
| **匯入 LZURLSTR** | 可貼 `https://artale.app/boss?data=...` 完整網址（會自動取 `data`）或純 LZ 字串；解壓後若為 `{ 鍵: [{ name, channel, timestamp }] }` 會依 **BOSS 顯示名稱**對應 `bossConfig` 並換算重生時間後合併。 |
| **清除所有紀錄** | 清空本地全部擊殺紀錄（會確認）。 |
| **快捷鍵** | 不在輸入框內時：主鍵盤 `+`／`=` 或數字鍵盤 `+` 開新增視窗；頻道輸入框開啟後可用 `numpad+` / `numpad-` 或 `[ ]` 切換 Boss；頻道欄可 **Enter** 送出。 |
| **資料儲存** | 存在瀏覽器 **本機**（換電腦或清快取可能遺失，重要資料請用分享備份）。 |

## 本機開發

```bash
npm install
npm run dev
```

## 部署到 GitHub

1. 到 [建立新 repository](https://github.com/new)（不要勾選自動加入 README，避免與本機歷史衝突）。
2. 在本專案根目錄執行（請替換 `<帳號>`、`<專案名>`）：

```bash
git remote add origin https://github.com/<帳號>/<專案名>.git
git push -u origin main
```

推送時若要求登入，請使用 [Personal Access Token (classic)](https://github.com/settings/tokens) 作為密碼，或使用 GitHub Desktop / SSH。

## 部署到 Vercel

1. 登入 [Vercel Dashboard](https://vercel.com/dashboard)。
2. **Add New… → Project**，選取上述 GitHub repository。
3. Framework 選 **Vite**（通常會自動偵測）；Build：`npm run build`；Output Directory：`dist`（與根目錄 `vercel.json` 一致）。
4. 點 **Deploy**。之後對預設分支的 push 會觸發自動建置。
