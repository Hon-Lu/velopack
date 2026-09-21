# Fork apps

自用分支專屬的檔案，上游沒有這個檔名，同步上游時不會撞到。

記錄每個使用這個 fork 打包的應用程式，以及它該帶的參數。

## 這個 fork 改了什麼

| 分支 | 改動 |
|---|---|
| `feat/stable-stub` | 新增 `vpk pack --stableStub`。stub 照常沿用主程式的資源（圖示、資訊清單、公司名、產品名、描述），但把版本欄位凍結成 `1.0.0`，讓它的位元組不再每次發版都變 |
| `feat/1060-no-stub` | 新增 `vpk pack --noStub`，完全不產生 stub。已送 PR 給上游（[velopack#1062](https://github.com/velopack/velopack/pull/1062)），本地不一定要用 |

### 為什麼需要 `--stableStub`

免安裝包根目錄那顆啟動器沒有數位簽章，而 `vpk pack` 預設會把主程式的**整棵資源樹**複製進去 —— 其中的版本欄位每次發版都變，所以它每一版都是一顆全世界沒見過的新檔案，永遠累積不到 Windows Defender 的檔案信譽，於是三不五時被報 `Trojan:Win32/Wacatac.B!ml`。

`--stableStub` 把版本欄位凍結之後，那顆檔在換圖示之前都是同一個雜湊，信譽得以累積。做法與 `Update.exe` 一致（它本來就只有「vendor 二進位 + 圖示」，所以雜湊本來就穩定）。

被凍結的欄位是**所有鍵名含 `version` 的**，因為 .NET 除了 `FileVersion` / `ProductVersion` 還會另外寫一個 `Assembly Version`，而 `ProductVersion` 裡甚至帶著 commit SHA。

### 什麼情況下雜湊還是會重算

stub 的內容是「vendor 的 stub 二進位」加上「主程式 exe 的整棵資源樹，版本欄位除外」，所以只要這兩者之一變動就會重算。

| 觸發 | 會重算嗎 |
|---|---|
| 換應用程式圖示 | 是 |
| 改 `app.manifest` | 是 |
| 改 `AssemblyCompany` / `AssemblyProduct` / `AssemblyDescription` / 著作權等版本資源字串 | 是 |
| 換 vpk / Velopack 版本（vendor 的 stub 本體換了） | 是 |
| 改版號、改 commit | 否，已凍結 |
| 改 `--packTitle` / `--packAuthors` | 否，只影響 stub 的**檔名**，不影響內容 |

平常發版只動版號與 commit，所以不會重算。

### 這個旗標沒有涵蓋的東西

- **`Update.exe`** 不受這個旗標影響。它本來就是「vendor 二進位 + 圖示」，雜湊本來就穩定 —— 但**換圖示一樣會讓它重算**。所以換 logo 的代價是根目錄那兩顆檔同時重來。
- **`Setup.exe`** 每次打包都內嵌整包 nupkg，內容必然隨版本改變，雜湊一定變，沒有辦法穩定化。

### 導入時的一次性過渡

發出第一個帶 `--stableStub` 的版本時，會產生一顆與舊版不同的全新 stub。既有使用者更新後拿到它，從那之後才定住 —— 也就是說導入當下反而是檔案信譽最低的時刻。這是一次性的，但要預期。

### 既有使用者會怎樣

`Update.exe` 在每次套用更新時都會從套件裡重新解出 stub 並覆寫根目錄那顆。覆寫動作本身照做（檔案時間會更新），但寫進去的位元組相同，所以使用者硬碟上不會出現新雜湊的檔案。發出第一個帶 `--stableStub` 的版本之後，既有使用者在下次更新時就會換成穩定版，不需要重新安裝。

### 如果之後拿到代碼簽章

簽章會改變檔案內容，雜湊會再變一次，而且時戳讓每次簽出來的結果都不同。屆時靠的是簽章者信譽而不是單一檔案的信譽，這個旗標的價值會下降 —— 但沒有壞處，可以繼續留著。

### 實測結果

以 OverTranslate 2.4.0 的 apphost 與把版本改成 2.5.0 的副本（其餘位元組不動）作為輸入，四條路徑得到同一個雜湊：

| 路徑 | 結果 |
|---|---|
| 免安裝包裡的 stub | `7FA6A84F…` |
| `Setup.exe` 安裝後根目錄的 stub | `7FA6A84F…` |
| 免安裝資料夾套用更新後 | `7FA6A84F…` |
| 安裝版套用更新後 | `7FA6A84F…` |

不帶旗標的對照組則是 2.4.0 `AB72693D…`、2.5.0 `19C4AEB6…`，每版一顆新檔。

## 怎麼用這個 fork 的 vpk

CI 原本是 `dotnet tool install -g vpk --version 1.2.0`，改成從這個 fork 建置並安裝：

```bash
git clone -b feat/stable-stub https://github.com/asd880921/velopack
cd velopack
cargo build --features windows          # 產生 stub / setup / update 二進位（Windows 上必須帶 --features windows）
dotnet pack src/vpk/Velopack.Vpk -c Release -o ./nupkg
dotnet tool install -g vpk --add-source ./nupkg --version <產出的版本>
```

需要 .NET SDK 與 Rust toolchain。

---

## OverTranslate

- Repo: <https://github.com/asd880921/OverTranslate>
- 打包腳本：`publish-velopack.ps1`
- 應帶參數：**`--stableStub`**

```
vpk pack `
    --packId OverTranslate `
    --packVersion <version> `
    --packDir <publishDir> `
    --mainExe OverTranslate.exe `
    --packTitle OverTranslate `
    --packAuthors Hon.Lu `
    --icon src/OverTranslate/icons/app.ico `
    --channel win `
    --outputDir artifacts/releases `
    --stableStub
```

備註：

- 相關 issue：[OverTranslate#210](https://github.com/asd880921/OverTranslate/issues/210)（Wacatac.B!ml 誤判處理計畫）
- `--stableStub` 之後，根目錄兩顆檔平常發版都不再變動：`Update.exe` 只在換圖示或換 vpk 版本時重算，`OverTranslate.exe`（stub）再加上改 `app.manifest` 或版本資源字串（見上面的表）
- 這個程式的 `app.manifest` 沒有要求提權（預設 `asInvoker`），只設定 DPI，stub 沿用它不會有副作用

---

## 新增其他應用時

照上面的格式加一段就好：應用名稱、repo 連結、打包腳本位置、該帶的參數、以及任何該應用特有的注意事項。
