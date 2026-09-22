# Fork apps

自用分支專屬的檔案，上游沒有這個檔名，同步上游時不會撞到。
記錄每個用這個 fork 打包的應用程式，以及它該帶的參數。

## 這條分支

`fork/stable-stub-1.2.0`，基於 **`1.2.0` tag**（不是 `develop`），只加一個 `vpk pack --stableStub`。

stub 照常沿用主程式的資源（圖示、資訊清單、公司名、產品名、描述），但把**所有鍵名含 `version` 的欄位**凍結成 `1.0.0`，讓它的位元組不再每次發版都變，以便累積 Windows Defender 的檔案信譽。

> 不只 `FileVersion` / `ProductVersion`：`ProductVersion` 裡帶著 commit SHA，而 .NET SDK 另外會寫一個 `Assembly Version`。漏掉後者只會差 1 個位元組，但雜湊仍然不同。

基於 1.2.0 而非 `develop`，是因為 develop 多出的 148 個 commit 含 Rust 更新器改動，會讓 `Update.exe` 的雜湊也一起變 —— 那顆檔的信譽會歸零，與本分支的目的相反。1.2.0 之後最有價值的上游修正（[velopack#1008](https://github.com/velopack/velopack/issues/1008)）實測對 OverTranslate 沒有影響。

## 什麼會讓雜湊重算

stub 的內容 =「vendor 的 stub 二進位」+「主程式 exe 的整棵資源樹，版本欄位除外」。

| 觸發 | 會重算嗎 |
|---|---|
| 換應用程式圖示 | 是 |
| 改 `app.manifest` | 是 |
| 改 `AssemblyCompany` / `AssemblyProduct` / `AssemblyDescription` / 著作權等字串 | 是 |
| 換 vendor 的 stub 二進位 | 是 |
| 改版號、改 commit | 否 |
| 改 `--packTitle` / `--packAuthors` | 否，只影響檔名 |

平常發版只動版號與 commit，所以不會重算。

## 其他要知道的

- **`Update.exe` 不受這個旗標影響**，它本來就穩定 —— 但**換圖示會讓它一起重算**。換 logo 的代價是根目錄那兩顆檔同時重來。
- **`Setup.exe` 無法穩定化**，它每次都內嵌整包 nupkg。
- **導入時有一次性過渡**：第一個帶旗標的版本會產生一顆全新 stub，從那之後才定住。
- **既有使用者不用重裝**：更新時 `Update.exe` 照常覆寫根目錄那顆，但寫進去的位元組相同。
- **拿到代碼簽章之後**，簽章會讓雜湊再變一次且每次簽都不同；屆時靠簽章者信譽，這個旗標價值下降但無害。

## 怎麼用

> **vendor 的 Rust 二進位必須沿用官方 1.2.0 的那份，不要自己編。**
> 本機或 CI 編出來的 `update.exe` / `stub.exe` / `setup.exe` 不會與官方 CI 的產物位元組相同（rustc 版本、build 組態都不同），那樣 `Update.exe` 的雜湊照樣會變，整件事就白做了。

因此**不需要 Rust toolchain**，只要 .NET SDK。把官方二進位放進 fork 的 `vendor/`，該路徑在 Debug 與 Release 下都會被搜尋：

```pwsh
# 1. 裝官方 1.2.0，只為了取得它的 vendor 二進位
dotnet tool install -g vpk --version 1.2.0
$vendor = "$env:USERPROFILE\.dotnet\tools\.store\vpk\1.2.0\vpk\1.2.0\vendor"

# 2. 取得 fork，把官方二進位放進去
#    不能用 --depth 1，見下面「clone 的兩個限制」。
git -c core.longpaths=true clone --single-branch -b fork/stable-stub-1.2.0 https://github.com/asd880921/velopack velopack-fork
Copy-Item "$vendor\*" velopack-fork\vendor -Recurse -Force

# 3. 建置
dotnet build velopack-fork\src\vpk\Velopack.Vpk -c Release -f net10.0
```

打包時不用 `vpk` 指令，直接跑建置產物就好 —— Release 下 vendor 是從執行檔往上三層找的，所以它看得到上面複製進去的官方二進位：

```pwsh
velopack-fork\build\Release\net10.0\vpk.exe pack <參數>
```

（`dotnet run --project velopack-fork\src\vpk\Velopack.Vpk -c Release --framework net10.0 --no-build -- pack <參數>` 也可以，只是每次都要多跑一次 MSBuild 評估。）

### clone 的兩個限制

- **不能 `--depth 1`**。這個倉用 Nerdbank.GitVersioning 算版號，它要走 commit 歷史，淺 clone 會讓建置直接失敗：`Shallow clone lacks the objects required to calculate version height`。用 `--single-branch` 完整 clone 即可（實測 17 秒、543 MB）。
- **`-c core.longpaths=true`**。`test/fixtures/packages/xunit.abstractions.2.0.0-beta-build2700/lib/portable-net45+win+wpa81+wp80+monotouch+monoandroid` 這串在目的地路徑稍深時就會 `Filename too long`，git 會 clone 成功但 checkout 失敗。那種半套的工作目錄裡 `vendor/` 不存在，接著 `Copy-Item "$vendor\*" <dst> -Recurse` 會把 `signing\` 與 `wix\` 的內容攤平成同一層 —— 攤平後 vpk 仍然打得出包，只是簽章與 wix 那些路徑會悄悄失準。

執行測試才需要 Rust（`cargo build --features windows` 產生 `testapp.exe`）。

**執行期函式庫不用換**：這個 fork 只動打包工具（`src/vpk/…`），`src/lib-csharp` 一行沒改，應用程式的 `<PackageReference Include="Velopack" Version="1.2.0" />` 維持用官方 NuGet。本分支建出來的 vpk 自報 `1.2.<height>-g<sha>`——Nerdbank.GitVersioning 把超出 1.2.0 tag 的 commit 數算進版號，所以這條分支每多一個 commit（包含只改這份文件）就 +1。打包時因此會看到這一條：

```
[WRN] Velopack library version is lower than vpk version (1.2.0.0 < 1.2.3.0). This can occasionally cause compatibility issues.
```

只是警告。實測 vpk 從 `1.2.2` 變成 `1.2.3` 之後，stub 與 `Update.exe` 的雜湊完全沒動——**vpk 自己的版號不會進到產物裡**。

---

## OverTranslate

- Repo: <https://github.com/asd880921/OverTranslate>
- 打包腳本：`publish-velopack.ps1`（CI 觸發，見 `.github/workflows/release.yml`）
- 相關 issue：[OverTranslate#210](https://github.com/asd880921/OverTranslate/issues/210)
- 應帶參數：**`--stableStub`**

```
pack `
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

這個程式的 `app.manifest` 沒有要求提權（預設 `asInvoker`），只設定 DPI，stub 沿用它不會有副作用。

### 實測一：fork 這邊的探針

輸入為 2.4.0 的真實 apphost，與把版本改成 2.5.0 的副本：

| 項目 | 雜湊（前 24 碼） | |
|---|---|---|
| 2.4.0 stub，不帶旗標 | `0B231877AA0DB19DA9265C21` | 與實際發布的 2.4.0 相同 |
| 2.4.0 stub，帶旗標 | `39FFA5EE4600EDFBB7A677C1` | |
| 2.5.0 stub，帶旗標 | `39FFA5EE4600EDFBB7A677C1` | 跨版本相同 |
| `Update.exe` | `9A1E419468148E96DD396D49` | 與實際發布的 2.4.0 相同 |

### 實測二：OverTranslate 端跑完整打包流程（2026-09-22）

真的改 csproj 版號重建（2.4.0 → 2.5.0）、換 commit（改 `InformationalVersion`）、換 `--packVersion`，以及改用「從 GitHub 全新 clone 後建置的 vpk」——四種情況打出來的都是同一顆：

| 檔案 | 大小 | SHA256 |
|---|---|---|
| 根目錄 stub `OverTranslate.exe` | 497,664 | `39ffa5ee4600edfbb7a677c1e5da3bfb3a2bb571a6db29a4926617f28bcedac0` |
| `Update.exe` | 3,971,072 | `9a1e419468148e96dd396d4935b348e3cb1ee67f2ecb437bbc112879dda36889` |

- nupkg 裡的 `lib/app/OverTranslate_ExecutionStub.exe` 與免安裝包根目錄是**同一顆**，所以安裝版使用者也一起定住
- 本機 .NET SDK 10.0.401 建出的 apphost，與 CI 建的 2.4.0 apphost 產生同一顆 stub —— **SDK 版本不影響 stub**
- OverTranslate 的 CI 打包後會比對這兩個值（`check-release-hashes.ps1`），對不上只發 GitHub 警示、不擋發布

---

## 新增其他應用

照上面的格式加一段：應用名稱、repo 連結、打包腳本位置、該帶的參數、該應用特有的注意事項。
