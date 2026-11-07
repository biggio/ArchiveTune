# ArchiveTune (Personal Fork) - Antigravity 開發與移植指引手冊 (gemini.md)

本文件專為 **Antigravity AI 助理** 與開發者設計，記錄此專案的環境需求、分支架構、客製化功能實作細節、編譯與發布流程，以利在不同 Antigravity 環境或新機器上快速重建與接手維護。

---

## 1. 專案背景與倉庫架構

- **專案簡介**：ArchiveTune 是一款開源的 Android 音樂串流播放器（基於 ViMusic / InnerTune 架構優化）。
- **個人倉庫 (origin)**：`git@github.com:biggio/ArchiveTune.git`（主分支：`main`）
- **官方上游 (upstream)**：`https://github.com/rukamori/ArchiveTune.git`
- **版本規範**：
  - 採用 **`<upstream-version>-r<N>`** 獨立修訂版號架構（當前版本：`15.1.0-r1`，對應上游 `15.1.0`）。
  - **核心精神**：保留 upstream 主版本號（如 `15.1.0`），後綴個人修訂號（`-r1`, `-r2`），避免上游更新時產生版本合併衝突。
  - `versionCode` 採用 `(upstream_versionCode * 100) + revision`（例如上游 142 -> 本專案 `14201`），確保 Android 系統安裝升級時 versionCode 永遠單調遞增（14201 > 14101）。

---

## 2. 環境與相依依賴 (Environment Setup)

在全新環境或新主機上建立此專案，需滿足以下條件：

### A. 必要路徑與環境變數 (macOS 範例)
```bash
# 1. 指定 JDK 21 (推薦直接使用 Android Studio 內建的 JBR)
export JAVA_HOME="/Applications/Android Studio.app/Contents/jbr/Contents/Home"
export PATH="$JAVA_HOME/bin:$PATH"

# 2. 指定 Android SDK 路徑
export ANDROID_HOME="$HOME/Library/Android/sdk"

# 3. 確保 local.properties 包含 SDK 路徑
echo "sdk.dir=$ANDROID_HOME" > local.properties
```

### B. Git 倉庫與子模組初始化 (Submodules)
本專案高度依賴 Git 子模組（`core`, `lyrics`, `IconPack`, `morideobfuscator`），**Clone 專案後必須完整初始化子模組**：
```bash
git clone git@github.com:biggio/ArchiveTune.git
cd ArchiveTune
git submodule update --init --recursive

# 設定 upstream 遠端以利未來同步上游更動
git remote add upstream https://github.com/rukamori/ArchiveTune.git
```

---

## 3. 本專案核心客製化功能實作總覽

接手此專案時，需注意以下已實作的自訂功能，切勿在重構或同步上游時被意外覆蓋：

### ① 啟動時自動播放 (Auto-Play On Start)
- **功能描述**：冷啟動 App 後自動開始播放音樂。
- **播放選曲邏輯**：
  1. **第一優先**：恢復關閉 App 前最後停留在待播清單（Queue）中的歌曲與進度。
  2. **第二優先**：若無上次隊列，則從資料庫自動挑選：快速精選推薦 (Quick Picks) > 按讚喜愛的歌曲 (Liked Songs, 最常聽優先) > 音樂庫全部歌曲 (All Songs, 最常聽優先)。
- **核心代碼檔案**：
  - `app/src/main/kotlin/moe/rukamori/archivetune/constants/PreferenceKeys.kt`：`AutoPlayOnStartKey`
  - `app/src/main/kotlin/moe/rukamori/archivetune/ui/screens/settings/PlayerSettings.kt`：設定介面「啟動時自動播放」開關（預設開啟）。
  - `app/src/main/kotlin/moe/rukamori/archivetune/MainActivity.kt`：冷啟動監聽器 `LaunchedEffect(playerConnection, autoPlayOnStart)`，判定 queue 並呼叫播放。
  - `app/src/main/kotlin/moe/rukamori/archivetune/playback/MusicService.kt`：負責隊列狀態恢復與呼叫播放執行。

### ② 啟動播放第一首 3 秒平滑音量淡入 (Audio Fade-In)
- **功能描述**：App 自動啟動播放時，第一首歌曲強制進行 3 秒由 0% 平滑漸升至目標音量的淡入，避免突然的大音量嚇到使用者。
- **核心代碼檔案**：
  - `app/src/main/kotlin/moe/rukamori/archivetune/playback/MusicService.kt`：
    - `startFadeIn(durationMs = 3,000L)`：透過三次平滑曲線（smoothstep）控制 ExoPlayer 音量。
    - 加入 `isFadingIn` 保護旗標，防止 2,000ms 音量看門狗（Volume Watchdog）將正在淡入的音量強行重置為 100%。
  - `app/src/main/kotlin/moe/rukamori/archivetune/playback/PlayerConnection.kt`：擴充 `startPlaybackExplicitly(fadeIn = true)` 與 `playQueue(..., fadeIn = true)`。

### ③ GitHub Releases OTA 線上自動更新系統
- **功能描述**：App 內建檢查更新機制直接讀取 GitHub Releases，下載 APK 並呼叫原生安裝器。
- **客製化亮點**：
  1. **預設倉庫指向個人 Fork**：`app/build.gradle.kts` 中將 `githubOwner` 改為 `"biggio"`。
  2. **UI 自由切換更新來源**：在「設定 > 檢查更新」中新增來源切換卡片：
     - 個人分支（Fork - `biggio/ArchiveTune`）
     - 原作者官方（Official - `rukamori/ArchiveTune`）
     - 自訂倉庫（Custom - 任意 `owner/repo`）
  3. **智慧 APK 資產匹配 (Smart Asset Matching)**：`app/src/main/kotlin/moe/rukamori/archivetune/utils/Updater.kt` 支援相容 Debug / Release、Universal / ARM64 APK 匹配，避免原版固定檔名報 404 的問題。
  4. **未知來源安裝權限引導**：`app/src/main/kotlin/moe/rukamori/archivetune/utils/AppUpdateInstaller.kt` 在 Android 8.0+ 自動引導開啟 `ACTION_MANAGE_UNKNOWN_APP_SOURCES` 權限。
  5. **專屬發布工作流**：`.github/workflows/ota-release.yml` 支援推送標籤 `v*` 自動打包發布 Universal APK。

### ④ 獨立修訂版號比較器 (`15.0.0-r<N>`)
- **功能描述**：官方原版 SemVer 僅比對 3 段式 `X.Y.Z`，若使用 `-r1` 會被誤判為 Pre-release 而無法從 `15.0.0` 升級。
- **核心代碼檔案**：
  - `app/src/main/kotlin/moe/rukamori/archivetune/utils/Updater.kt`：
    - 正則式升級：`Regex("""(?i)\bv?(\d+)\.(\d+)\.(\d+)(?:(?:\.|\-r)(\d+))?(?:-([0-9A-Za-z.-]+))?(?:\+[0-9A-Za-z.-]+)?\b""")`
    - `SemVer` 結構新增 `revision: Int = 0`，支援精確判定 `15.0.0-r1 > 15.0.0` 與 `15.0.0-r2 > 15.0.0-r1`，並完全相容未來的上游更新（`15.0.1 > 15.0.0-r<N>`）。

### ⑤ libquickjs.so 閃退熱修復 (Hotfix: YouTube Player Script Crash) [已隨上游修復解除]
- **問題背景**：YouTube 於 2026 年 9 月更換播放器 `base.js`，引入深層遞迴陣列轉換，導致內嵌的 QuickJS 引擎原生執行緒 Stack 耗盡爆掉（`libquickjs.so` SIGSEGV 閃退）。
- **處理現況**：
  - **上游官方已正式修復**：ArchiveTune upstream 官方已發布新版（v15.1.0 及更新 commit），在官方的 `morideobfuscator` 中已整合 `WORKER_STACK_SIZE_BYTES` 擴展與防禦遞迴陣列轉換溢出機制（參見 upstream commit `f332845` 與 `51816ad`）。
  - **本版本解除熱修復**：我們不再需要手動鎖定 commit 或自行維護該熱修復，直接對齊並同步 upstream 最新版本子模組。

### ⑥ CI / GitHub Actions 簽署備援修復
- **問題背景**：原版 `.github/workflows/build.yml` 和 `release.yml` 依賴第三方 Action 並強制要求 `secrets.KEYSTORE`，在個人 Fork 執行時必會報錯 `Cannot find signingKey/ANDROID_SIGNING_KEY`。
- **修復方案**：
  - 改用 Android SDK 原生 `apksigner`。
  - 當偵測不到 `secrets.KEYSTORE` 時，自動在 runner 生成 `debug.keystore` 進行有效簽署，確保 CI 建置 100% 成功。

---

## 4. 常見操作指令 (Commands Cheat Sheet)

### 本地編譯除錯版本 (Debug APK)
```bash
export JAVA_HOME="/Applications/Android Studio.app/Contents/jbr/Contents/Home"
export ANDROID_HOME="$HOME/Library/Android/sdk"
export PATH="$JAVA_HOME/bin:$PATH"

./gradlew assembleGmsMobileUniversalDebug -x lint
# 產出路徑：app/build/outputs/apk/gmsMobileUniversal/debug/app-gms-mobile-universal-debug.apk
```

### 本地編譯發布版本 (Release APK)
```bash
./gradlew assembleGmsMobileUniversalRelease -x lint
# 產出路徑：app/build/outputs/apk/gmsMobileUniversal/release/app-gms-mobile-universal-release-unsigned.apk
```

### 代碼格式化 (Spotless)
若編譯時遇到 Spotless / KtLint 格式報錯：
```bash
./gradlew spotlessApply
```

### 如何發布新的 OTA 更新 (例如發布 15.1.0-r1)
1. 修改 `app/build.gradle.kts`：
   ```kotlin
   versionCode = 14201
   versionName = "15.1.0-r1"
   ```
2. 提交代碼並推送：
   ```bash
   git add app/build.gradle.kts
   git commit -m "bump: 15.1.0-r1"
   git push origin main
   ```
3. 打上 Tag 並推送：
   ```bash
   git tag v15.1.0-r1
   git push origin v15.1.0-r1
   ```
4. GitHub Actions 的 `ota-release.yml` 會自動編譯 Universal APK 並發布至 GitHub Releases，使用者手機將於「設定 > 檢查更新」自動收到更新提示！

---

## 5. 檔案結構速查

- `app/build.gradle.kts`：App 版本代碼、發布者名稱、Flavor 與簽署設定。
- `app/src/main/kotlin/.../playback/MusicService.kt`：播放核心服務、淡入控制、隊列持久化。
- `app/src/main/kotlin/.../playback/PlayerConnection.kt`：UI 與 Service 溝通橋樑。
- `app/src/main/kotlin/.../MainActivity.kt`：冷啟動自動播放判定與隊列恢復。
- `app/src/main/kotlin/.../utils/Updater.kt`：OTA 版本比較、API 請求、快取管理。
- `app/src/main/kotlin/.../utils/AppUpdateInstaller.kt`：APK 下載與系統安裝器呼叫。
- `app/src/main/kotlin/.../ui/screens/settings/UpdateScreen.kt`：更新介面與更新來源切換。
- `morideobfuscator/`：子模組，包含 YouTube 播放器解密與 QuickJS 執行環境。
- `.github/workflows/ota-release.yml`：專屬 OTA 自動化打包發布工作流。
