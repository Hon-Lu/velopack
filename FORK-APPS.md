# Fork apps

自用分支專屬的檔案，上游沒有這個檔名，同步上游時不會撞到。

記錄每個使用這個 fork 打包的應用程式，以及它該帶的參數。

## 這條分支是什麼

`fork/stable-stub-1.2.0`，**基於 `1.2.0` tag**（不是 `develop`），只加一個 `vpk pack --stableStub`。

stub 照常沿用主程式的資源（圖示、資訊清單、公司名、產品名、描述），但把版本欄位凍結成 `1.0.0`，讓它的位元組不再每次發版都變。

### 為什麼基於 1.2.0 而不是 develop

`develop` 比 1.2.0 多 148 個 commit，其中包含 Rust 更新器的改動。若基於 develop：

- 出廠的 `Update.exe` 是 develop 期的建置，**雜湊會變** —— 那顆檔目前的信譽會歸零，與本分支的目的相反
- vpk 版本會是 1.2.149，而 app 引用的 `Velopack` NuGet 是 1.2.0，`CompatUtil` 每次打包都會警告版本不符
- 一次引入 148 個無關變動，出問題時難以歸因

基於 1.2.0 則只有一個變數。已確認 1.2.0 之後最有價值的上游修正（[velopack#1008](https://github.com/velopack/velopack/issues/1008) delta 失效）**沒有影響到 OverTranslate** —— 實測 2.4.0 的 delta 包裡沒有任何 `.bsdiff` 成員。

### 被凍結的欄位

**所有鍵名含 `version` 的**，不只 `FileVersion` / `ProductVersion`：

- `ProductVersion` 裡帶著 commit SHA，所以每個 commit 都會變
- .NET SDK 另外會寫一個 `Assembly Version`，漏掉它就只差 1 個位元組但雜湊仍然不同

### 什麼情況下雜湊還是會重算

stub 的內容是「vendor 的 stub 二進位」加上「主程式 exe 的整棵資源樹，版本欄位除外」。

| 觸發 | 會重算嗎 |
|---|---|
| 換應用程式圖示 | 是 |
| 改 `app.manifest` | 是 |
| 改 `AssemblyCompany` / `AssemblyProduct` / `AssemblyDescription` / 著作權等版本資源字串 | 是 |
| 換 vendor 的 stub 二進位 | 是 |
| 改版號、改 commit | 否，已凍結 |
| 改 `--packTitle` / `--packAuthors` | 否，只影響 stub 的**檔名**，不影響內容 |

平常發版只動版號與 commit，所以不會重算。

### 這個旗標沒有涵蓋的東西

- **`Update.exe`** 不受這個旗標影響。它本來就是「vendor 二進位 + 圖示」，雜湊本來就穩定 —— 但**換圖示一樣會讓它重算**。所以換 logo 的代價是根目錄那兩顆檔同時重來。
- **`Setup.exe`** 每次打包都內嵌整包 nupkg，內容必然隨版本改變，沒有辦法穩定化。

### 導入時的一次性過渡

發出第一個帶 `--stableStub` 的版本時，會產生一顆與舊版不同的全新 stub。既有使用者更新後拿到它，從那之後才定住 —— 導入當下反而是檔案信譽最低的時刻。這是一次性的，但要預期。

### 既有使用者會怎樣

`Update.exe` 在每次套用更新時都會從套件裡重新解出 stub 並覆寫根目錄那顆。覆寫動作照做（檔案時間會更新），但寫進去的位元組相同，所以硬碟上不會出現新雜湊的檔案。不需要重新安裝。

### 如果之後拿到代碼簽章

簽章會改變檔案內容，雜湊會再變一次，而且時戳讓每次簽出來的結果都不同。屆時靠的是簽章者信譽而不是單一檔案的信譽，這個旗標的價值會下降 —— 但沒有壞處，可以繼續留著。

## 怎麼用這個 fork 的 vpk

> **關鍵：vendor 的 Rust 二進位必須沿用官方 1.2.0 的那份，不要自己編。**
> 本機編出來的 `update.exe` / `stub.exe` / `setup.exe` 與官方 CI 的產物不會位元組相同（rustc 版本、debug/release 都不同），那樣 `Update.exe` 的雜湊照樣會變，整件事就白做了。

因此**不需要 Rust toolchain**，只要 .NET SDK：

```bash
# 1. 先取得官方 1.2.0 的 vendor 二進位
dotnet tool install -g vpk --version 1.2.0
#    位置：~/.dotnet/tools/.store/vpk/1.2.0/vpk/1.2.0/vendor/

# 2. 建置這個 fork 的 vpk
git clone -b fork/stable-stub-1.2.0 https://github.com/asd880921/velopack
cd velopack
mkdir -p target/debug
cp <官方 vendor 路徑>/update.exe <官方 vendor 路徑>/stub.exe <官方 vendor 路徑>/setup.exe target/debug/
dotnet build src/vpk/Velopack.Vpk

# 3. 打包時直接用這個專案
dotnet run --project src/vpk/Velopack.Vpk --framework net10.0 --no-build -- pack ...
```

若要打包成 global tool 安裝，`dotnet pack src/vpk/Velopack.Vpk` 之後要確認產出的工具包裡 `vendor/` 是官方那份。

執行測試才需要 Rust（`cargo build --features windows` 產生 `testapp.exe`）。

### 版本相容

本分支建出來的 vpk 報 `1.2.0-gf2edcbc`，與 app 引用的 `Velopack` NuGet 1.2.0 相符，`CompatUtil` 不會跳版本警告。

**執行期函式庫不用換**：這個 fork 只動打包工具（`src/vpk/…`），`src/lib-csharp` 一行沒改，所以應用程式的 `<PackageReference Include="Velopack" Version="1.2.0" />` 維持用 nuget.org 的官方套件。

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
- 這個程式的 `app.manifest` 沒有要求提權（預設 `asInvoker`），只設定 DPI，stub 沿用它不會有副作用

### 實測結果

輸入是 OverTranslate 2.4.0 的真實 apphost，以及把版本改成 2.5.0 的副本（其餘位元組不動）：

| 項目 | 雜湊（前 24 碼） | 說明 |
|---|---|---|
| 2.4.0 stub，不帶旗標 | `0B231877AA0DB19DA9265C21` | **與實際發布的 2.4.0 相同**，證明建置環境忠實重現發版流程 |
| 2.4.0 stub，帶旗標 | `39FFA5EE4600EDFBB7A677C1` | |
| 2.5.0 stub，帶旗標 | `39FFA5EE4600EDFBB7A677C1` | 跨版本相同 |
| `Update.exe` | `9A1E419468148E96DD396D49` | **與實際發布的 2.4.0 相同**，使用者手上那顆不會被換掉 |

---

## 新增其他應用時

照上面的格式加一段就好：應用名稱、repo 連結、打包腳本位置、該帶的參數、以及任何該應用特有的注意事項。
