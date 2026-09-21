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

- 換應用程式圖示
- 換 vpk / Velopack 版本（vendor 的 stub 二進位本身換了）
- 改 `--packTitle`、`--packAuthors`，或主程式的資訊清單、描述、著作權等欄位

前兩項是預期內的；最後一項平常不會動。

### 既有使用者會怎樣

`Update.exe` 在每次套用更新時都會從套件裡重新解出 stub 並覆寫根目錄那顆，所以發出第一個帶 `--stableStub` 的版本之後，既有使用者在下次更新時就會換成穩定版的 stub，從那之後就不再變動。不需要他們重新安裝。

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
- `--stableStub` 之後，免安裝包根目錄的 `OverTranslate.exe` 與 `Update.exe` 兩顆的雜湊都只在換圖示時才變
- 這個程式的 `app.manifest` 沒有要求提權（預設 `asInvoker`），只設定 DPI，stub 沿用它不會有副作用

---

## 新增其他應用時

照上面的格式加一段就好：應用名稱、repo 連結、打包腳本位置、該帶的參數、以及任何該應用特有的注意事項。
