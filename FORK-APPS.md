# Fork apps

自用分支專屬的檔案，上游沒有這個檔名，同步上游時不會撞到。
記錄每個用這個 fork 打包的應用程式，以及它該帶的參數。

## 這條分支

`fork/no-stub-1.2.0`，基於 **`1.2.0` tag**（不是 `develop`），加了兩個 `vpk pack` 旗標：

| 旗標 | 作用 | 現在用哪個 |
|---|---|---|
| `--noStub` | 完全不產生啟動器 stub，套件與免安裝包都不含它 | **這個** |
| `--stableStub` | 照常產生 stub，但凍結它的版本欄位，讓它跨版本位元組不變 | 留著當退路，沒在用 |

兩個旗標針對的是同一件事：根目錄那顆未簽章的啟動器 stub 是 Windows Defender 報
`Trojan:Win32/Wacatac.B!ml` 的主要來源。`--stableStub` 是讓它至少累積得到檔案信譽，
`--noStub` 則是直接讓它不存在 —— 實測前者不夠（公司電腦下載免安裝版、解壓當下就被攔），
所以現在走後者。

### `--noStub` 為什麼有效

關鍵不在「免安裝包的根目錄少一顆檔」，而在**它同時不進 `.nupkg`**：

更新器每次套用更新都會把套件裡的 stub 解回根目錄（`Bundle.extract_stubs_to_dir`），而它只解
「套件裡有的」。套件裡沒有它，就沒有東西可以還原 —— 而且**現場的舊版 `Update.exe` 不必更新
也會照這個規則走**，因為 1.2.0 與 develop 的這段 Rust 完全相同。

所以 vendor 的二進位一顆都不用換，`Update.exe` 的雜湊不受影響。

### 與上游 PR 的關係

`--noStub` 已經以 [velopack#1060](https://github.com/velopack/velopack/issues/1060) 送 PR 回上游，
那條分支是 `feat/1060-no-stub`，**基於 `develop`**（上游要求）。**不要拿它來打包**：

- vpk 版號會是 1.2.15x，而應用程式引用的 `Velopack` 函式庫是 1.2.0，delta 與 manifest 的相容性沒人驗過
- develop 比 1.2.0 多 148 個 commit，含 Rust 更新器改動；出廠的 `Update.exe` 換成 develop 期的建置，那顆檔的信譽會歸零

這條分支是同一份改動在 1.2.0 上的移植。程式碼差異只有兩處：develop 的 `GetStubBaseName()` 在
1.2.0 是 `Path.GetFileNameWithoutExtension`，以及 1.2.0 還沒有 `WindowsPackOptionsValidator`，
所以 `--noStub` 與 `--msi` 的互斥改成在 runner 裡丟 `UserInfoException`。

1.2.0 之後最有價值的上游修正（[velopack#1008](https://github.com/velopack/velopack/issues/1008)）
實測對 OverTranslate 沒有影響。

## 拿掉 stub 之後要知道的事

- **安裝版不受影響**：捷徑本來就指向 `current\<主程式>`（`VelopackLocator::get_main_exe_path`），不是指向 stub。
- **免安裝版的使用者會找不到入口**：root 只剩 `Update.exe`、`current\` 與 `.portable`。雙擊 `Update.exe`
  只會印 `No known subcommand was used`，不能當入口。OverTranslate 的做法是在打包腳本裡補一個
  相對路徑的 `.lnk`，見下面那一節。
- **既有安裝版會留下一顆孤兒 stub**：更新器只解出套件裡有的 stub，不會刪掉硬碟上已經存在的那顆。
  它還能用（它就是去啟動 `current\` 的主程式），但**舊使用者的誤判來源不會因為這次改動而消失**。
- **`Setup.exe` 無法穩定化**，它每次都內嵌整包 nupkg。
- **`Update.exe` 還在**，內容是「vendor 的 update.exe + `--icon` 指定的圖示 + 打包時的簽章」，與版號、
  commit 無關。會讓它重算的只有三件事：換圖示、換 vpk 版本、換簽章金鑰。

## 怎麼用

> **vendor 的 Rust 二進位必須沿用官方 1.2.0 的那份，不要自己編。**
> 本機或 CI 編出來的 `update.exe` / `stub.exe` / `setup.exe` 不會與官方 CI 的產物位元組相同（rustc 版本、build 組態都不同），那樣 `Update.exe` 的雜湊照樣會變。

因此**不需要 Rust toolchain**，只要 .NET SDK（10.x，相依套件是 10.0 那一代）。把官方二進位放進 fork 的 `vendor/`，該路徑在 Debug 與 Release 下都會被搜尋：

```pwsh
# 1. 裝官方 1.2.0，只為了取得它的 vendor 二進位
dotnet tool install -g vpk --version 1.2.0
$vendor = "$env:USERPROFILE\.dotnet\tools\.store\vpk\1.2.0\vpk\1.2.0\vendor"

# 2. 取得 fork，把官方二進位放進去
#    不能用 --depth 1，見下面「clone 的兩個限制」。
git -c core.longpaths=true clone --single-branch -b fork/no-stub-1.2.0 https://github.com/asd880921/velopack velopack-fork
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

### 版本相容

**執行期函式庫不用換**：這個 fork 只動打包工具（`src/vpk/…`），`src/lib-csharp` 一行沒改，應用程式的 `<PackageReference Include="Velopack" Version="1.2.0" />` 維持用官方 NuGet。

本分支建出來的 vpk 自報 `1.2.<height>-g<sha>`——Nerdbank.GitVersioning 把超出 1.2.0 tag 的 commit 數算進版號，所以這條分支每多一個 commit（包含只改這份文件）就 +1。打包時因此會看到這一條：

```
[WRN] Velopack library version is lower than vpk version (1.2.0.0 < 1.2.4.0). This can occasionally cause compatibility issues.
```

只是警告。實測 vpk 版號變動之後，產物的雜湊完全沒動——**vpk 自己的版號不會進到產物裡**。

---

## OverTranslate

- Repo: <https://github.com/asd880921/OverTranslate>
- 打包腳本：`publish-velopack.ps1`（CI 觸發，見 `.github/workflows/release.yml`）
- 相關 issue：[OverTranslate#210](https://github.com/asd880921/OverTranslate/issues/210)
- 應帶參數：**`--noStub`**

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
    --noStub `
    --signParams "/sha1 <指紋> /fd SHA256"
```

簽章用的是一張自簽憑證（`CN=Hon.Lu, O=OverTranslate`，RSA 4096，2049 到期）。
**刻意不加時戳**：時戳會讓相同內容每次簽出不同位元組，`Update.exe` 的雜湊就會每版重算。
vpk 會簽 packDir 裡所有 PE 檔，而已經帶有受信任簽章的檔（微軟簽的 .NET 執行檔）會自動跳過
（`CodeSign.ShouldSign` → `SignatureState.SignedAndTrusted`），不會被自簽蓋掉。

這個程式的 `app.manifest` 沒有要求提權（預設 `asInvoker`），只設定 DPI。

### 免安裝包的入口

`--noStub` 之後 root 沒有可點的東西，所以 `publish-velopack.ps1` 在 `vpk pack` 之後補一個
`OverTranslate.lnk` 進 zip 根目錄，指向 `current\OverTranslate.exe`。兩個要點：

- **必須帶相對路徑欄位**（`IShellLink::SetRelativePath`）。捷徑裡存的絕對路徑是打包機器上的位置，
  使用者機器上不存在，Windows 會退而用相對路徑去找。`WScript.Shell` 建的只有絕對路徑，
  解壓到別處就是死捷徑。
- **圖示在第一次點開之前是通用的**。Shell 取圖示時不走相對路徑；點過一次之後 Windows 會把解析出來的
  路徑寫回捷徑，圖示就變成主程式的。已知且接受的代價。

免安裝 zip 不在任何校驗鏈裡（`releases.<channel>.json` 只記 nupkg 的 SHA256），所以打包後改它是安全的。

### 實測（2026-09-23，OverTranslate 端跑完整流程）

| 情境 | 結果 |
|---|---|
| 打包 | `Skipping launcher stub, --noStub was specified.`，簽章檔數 16 → 15 |
| 免安裝包 | root 只有 `.portable`、`Update.exe`、`OverTranslate.lnk`，整包 0 個 `_ExecutionStub` |
| `.nupkg` | 0 個 `_ExecutionStub` |
| 從「有 stub 的舊版」更新上來 | 舊 stub 原地不動（時間戳沒變），**沒有產生新的** |
| 把舊 stub 刪掉再更新一次 | root 仍然只有 `Update.exe`、`current\`、`packages\`、`.portable` |
| 捷徑解壓到任意路徑 | 解析到 `<解壓位置>\current\OverTranslate.exe`（關掉 Shell 的搜尋補救仍然成立） |
| Defender 掃 zip 與解壓後資料夾 | 0 偵測 |

`Update.exe` 的基準雜湊（含自簽章）是
`ae4a116a15e5cda0e423e8fca5f1b02326b502553397d709fc6a7090bc958e9d`（3,973,288 bytes）。
OverTranslate 的 CI 每次打包後都會比對它，順便確認 stub 沒有跑回來。

---

## 新增其他應用

照上面的格式加一段：應用名稱、repo 連結、打包腳本位置、該帶的參數、該應用特有的注意事項。
