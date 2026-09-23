# Fork apps

自用分支專屬的檔案，上游沒有這個檔名，同步上游時不會撞到。
記錄每個用這個 fork 打包的應用程式，以及它該帶的參數。

## 這條分支

`fork/no-stub-1.2.0`，基於 **`1.2.0` tag**（不是 `develop`），加了兩個 `vpk pack` 旗標：

| 旗標 | 作用 | 現在用哪個 |
|---|---|---|
| `--noStub` | Velopack 不再產生啟動器 stub（要不要自己放一顆同名的進 packDir，是應用程式的事） | **這個** |
| `--stableStub` | 照常產生 stub，但凍結它的版本欄位，讓它跨版本位元組不變 | 留著當退路，沒在用 |

兩個旗標針對的是同一件事：根目錄那顆未簽章的啟動器 stub 是 Windows Defender 報
`Trojan:Win32/Wacatac.B!ml` 的主要來源。`--stableStub` 是讓它至少累積得到檔案信譽，
`--noStub` 則是直接讓它不存在 —— 實測前者不夠（公司電腦下載免安裝版、解壓當下就被攔），
所以現在走後者。

### `--noStub` 為什麼有效

關鍵不在「免安裝包的根目錄少一顆檔」，而在 packDir 裡那個位置**歸應用程式自己決定**了。

更新器每次套用更新都會把套件裡的 stub 解回根目錄（`Bundle.extract_stubs_to_dir`），而它只解
「套件裡有的」，解出來就覆蓋同名檔案。所以：

- 套件裡什麼都沒有 → 沒有東西可以還原，根目錄乾淨
- 應用程式自己放一顆同名的（`<主程式>_ExecutionStub.exe`）進 packDir → 更新器反過來變成**派送
  管道**，每一版都把它解回根目錄，連使用者手上那顆舊的 Velopack stub 一起覆蓋掉

兩種情況**現場的舊版 `Update.exe` 都不必更新**，因為 1.2.0 與 develop 的這段 Rust 完全相同。
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
  只會印 `No known subcommand was used`，不能當入口。OverTranslate 的做法是自己寫一顆啟動器頂替，
  見下面那一節。
- **孤兒 stub 有解**：更新器不會主動刪掉硬碟上已經存在的那顆，但只要應用程式放一顆同名的進 packDir，
  下一次更新就會把它覆蓋掉。什麼都不放的話，**舊使用者的誤判來源會一直留在硬碟上**。
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

`--noStub` 之後 root 沒有可點的東西。OverTranslate 自己用 Rust 寫了一顆啟動器（原始碼與編好的
二進位都在該倉的 `src/OverTranslate.Launcher/`），只做一件事：把 `current\OverTranslate.exe` 叫起來。

`publish-velopack.ps1` 在 `vpk pack` **之前**把它複製進 packDir，檔名用約定的
`OverTranslate_ExecutionStub.exe`，於是：

- vpk 會連同其他 PE 一起簽章，它也被打進 `.nupkg`
- 安裝與每一次更新，更新器把它解回安裝根目錄、改名成 `OverTranslate.exe`（**安裝版也因此有了
  可點的入口**，而且會覆蓋掉舊的 Velopack stub）
- 解 `current\` 的那條路徑會跳過 `*_ExecutionStub.exe`，所以 `current\` 不會多一份

免安裝包要另外處理兩件事，因為它不經過更新器：

- 根目錄那顆由打包腳本另外簽一份放進去（同一支 signtool、同樣不加時戳 → **與套件裡那顆位元組相同**）
- `CreatePortablePackage` 是把整個 packDir 複製進 zip 的 `current\`，原本會再把 stub 搬到根目錄，
  而 `--noStub` 把那個搬移跳掉了 —— 所以 `current\` 裡那份多餘的要由打包腳本刪掉

免安裝 zip 不在任何校驗鏈裡（`releases.<channel>.json` 只記 nupkg 的 SHA256），所以打包後改它是安全的。

> 先前試過相對路徑的 `.lnk`，**已否決**：捷徑一定會帶絕對路徑，跨機器解壓後 Windows 會重新對應，
> 實測在真實下載情境是死捷徑。`.cmd` 也否決了。

### 實測（2026-09-23，OverTranslate 端跑完整流程）

| 情境 | 結果 |
|---|---|
| 打包 | `Skipping launcher stub, --noStub was specified.` |
| 免安裝包 | root 是 `.portable`、`Update.exe`、應用自己的 `OverTranslate.exe`；整包 0 個 `_ExecutionStub` |
| `.nupkg` | `lib/app/OverTranslate_ExecutionStub.exe`，與免安裝包根目錄那顆**位元組相同** |
| 從「根目錄還是舊 Velopack stub」的安裝版更新上來 | log：`Extracting stub '…_ExecutionStub.exe' to '…\OverTranslate.exe'`，根目錄那顆被覆蓋；`current\` 沒有多出 `_ExecutionStub`（log：`Skipped Stub (obsolete)`） |
| 只換版號（1.6.1 → 1.6.2）再打一次 | 啟動器與 `Update.exe` 的雜湊完全相同 |
| Defender 掃 zip 與解壓後資料夾 | 0 偵測 |

`Update.exe` 的基準雜湊（含自簽章）是
`ae4a116a15e5cda0e423e8fca5f1b02326b502553397d709fc6a7090bc958e9d`（3,973,288 bytes），
啟動器是 `1dd826ad50481ec7bffa57ad9cafe7c6dad79541009129dca2d1e4266a7a76bf`（347,304 bytes）。
OverTranslate 的 CI 每次打包後都會比對，順便確認 Velopack 的 stub 沒有跑回來。

---

## 新增其他應用

照上面的格式加一段：應用名稱、repo 連結、打包腳本位置、該帶的參數、該應用特有的注意事項。
