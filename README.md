# Zorin OS Setting Notes
My Zorin OS configuration notes.

這個github專案是私人倉庫 用於記錄 Zorin OS 的各項設定筆記。我期望透過與 AI 協作，探索並實現「以自然語言修改系統設定」的目標。

# 設定相關

## Gemini CLI 安裝與認證指南

**Date**: June 7, 2026 at 04:22 PM
**Turns**: 1
**Source**: [Gemini Chat](https://gemini.google.com/app/ce85b0d4b224bb60)

---

#### Turn 1

###### 🤖 Assistant

```markdown
## 系統環境建置文件：Zorin OS 原生部署 Gemini CLI 

#### 1. 架構評估與依賴聲明
* **目標系統**：Zorin OS / Ubuntu 系發行版
* **核心依賴**：Node.js v20+ 
* **衝突排除**：
    * ❌ **嚴禁**使用 `sudo apt install nodejs npm` (版本過舊且易導致全域套件權限 EACCES 錯誤)
    * ❌ **嚴禁**使用 `sudo snap install gemini` (命名空間衝突，Snap 庫中為通訊協定瀏覽器，非 Google CLI)

---

#### 2. 部署流程 (NVM 隔離安裝法)

###### 2.1 環境初始化 (安裝 Node Version Manager)
```bash
## 1. 下載並執行 NVM 安裝腳本
curl -o- [https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh](https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh) | bash

## 2. 重新載入環境變數 (使 nvm 指令生效)
source ~/.bashrc

```

###### 2.2 核心依賴安裝 (Node.js v20 LTS)

```bash
## 1. 請求安裝 v20 版本
nvm install 20

## 2. 設定 v20 為系統預設版本
nvm use 20
nvm alias default 20

## 3. 狀態驗證 ✅ (預期輸出 Node v20.x.x 與 npm 10.x.x+)
node -v
npm -v

```

###### 2.3 核心應用安裝 (Gemini CLI)

```bash
## 透過 npm 進行全域安裝 (因 NVM 接管，無需 sudo 權限) 🚀
npm install -g @google/gemini-cli

```

#### 3. 系統驗證與授權

###### 3.1 啟動指令

```bash
gemini

```

###### 3.2 授權模式 (二擇一)

| 驗證機制 | 觸發條件 | 適用環境 | 系統狀態 |
| --- | --- | --- | --- |
| OAuth 2.0 | 終端機直接輸入 gemini | 具備 GUI 瀏覽器之桌面環境 | ✅ 建議一般開發使用 |
| API Key | export GEMINI_API_KEY="<金鑰>" | 無介面伺服器 (Headless) / CI 腳本 | ⚠️ 需綁定 Google Cloud / AI Studio 計費 |

###### 3.3 CLI 內建指令檢測
成功進入互動介面後，可輸入以下指令確認功能模組掛載狀態：

- `/model`：切換底層驅動模型 (如 Gemini 1.5 Pro / Flash)
- `/tools`：檢視可用之系統工具 (本地檔案讀寫、終端機執行等 MCP 伺服器)
- `/help`：輸出指令手冊

```

**[資料信心度：+9 定理級實證]**

```

---

*Exported from [Voyager](https://github.com/Nagi-ovo/gemini-voyager)*  
*Generated on June 7, 2026 at 04:22 PM*


## Zorin OS 雙 WAN 負載平衡 (ECMP) 建置 - Date: 2026-04-09 - Environment: Zorin OS / Bash Terminal

#### 🛠 Technical Review：
| Status | Method | Key Cause/Error |
| :--- | :--- | :--- |
| ✅ Success | 環境參數探測 (`ip route`) | 確認雙網段 (`192.168.50.0/24`、`172.20.10.0/28`) 無衝突，具備 ECMP 建置條件。 |
| ❌ Failure | 預期純負載平衡解決「網路瞬斷」 | Linux 原生 ECMP 採連線雜湊，無法掩蓋單一實體線路掉包導致的應用層斷線。 |
| ✅ Success | 暫時性寫入多路徑路由 | 成功驗證 `ip route replace default nexthop...` 寫入 RAM 並生效。 |
| ✅ Success | `NetworkManager-dispatcher` 自動化 | 解決網路重連或 DHCP 刷新時，自定義路由遭系統覆蓋失效之風險。 |

#### 📦 Dependencies and Configuration：
* 核心模組：`iproute2`、`NetworkManager`
* 有線介面 (WAN 1)：`enp6s0` (網段 `192.168.50.0/24`，閘道 `192.168.50.1`)
* 無線介面 (WAN 2)：`wlp5s0` (網段 `172.20.10.0/28`，閘道 `172.20.10.1`)
* 權限要求：`sudo` (root) 權限

### 🚀 Execution Script：
```bash
# ---------------------------------------------------------
# Step 1: 寫入 NetworkManager 自動化調度腳本 (解決重啟失效問題)
# ---------------------------------------------------------
sudo tee /etc/NetworkManager/dispatcher.d/99-dual-wan-ecmp > /dev/null << 'EOF'
#!/bin/bash
# 限制僅監聽特定網卡變動
if [[ "$1" == "enp6s0" || "$1" == "wlp5s0" ]]; then
    # 僅在網卡上線時觸發
    if [[ "$2" == "up" ]]; then
        # 檢查兩張網卡是否皆已取得各自網段的 IP
        if ip route | grep -q "192.168.50.0/24" && ip route | grep -q "172.20.10.0/28"; then
            # 覆寫為 ECMP 負載平衡路由
            ip route replace default \
                nexthop via 192.168.50.1 dev enp6s0 weight 1 \
                nexthop via 172.20.10.1 dev wlp5s0 weight 1
        fi
    fi
fi
EOF

# ---------------------------------------------------------
# Step 2: 賦予執行權限
# ---------------------------------------------------------
sudo chmod +x /etc/NetworkManager/dispatcher.d/99-dual-wan-ecmp

# ---------------------------------------------------------
# Step 3: 重啟網路服務載入設定
# ---------------------------------------------------------
sudo systemctl restart NetworkManager

# ---------------------------------------------------------
# Step 4: 最終狀態驗證 (應輸出包含雙 nexthop 的 default 路由)
# ---------------------------------------------------------
ip route show exact default
```



## solaar 羅技鍵鼠設定

```rules.yaml
%YAML 1.3
---
- Rule:
  - Key: [Emoji Smiley Heart Eyes, pressed]
  - KeyPress:
    - End
    - click
- Rule:
  - Key: [Emoji Crying Face, pressed]
  - KeyPress:
    - Insert
    - click
- Rule:
  - Key: [Emoji Smiley, pressed]
  - KeyPress:
    - KP_Multiply
    - click
- Rule:
  - Key: [Emoji Smilie With Tears, pressed]
  - KeyPress:
    - Page_Up
    - click
- Rule:
  - Key: [Emoji, pressed]
  - KeyPress:
    - Page_Down
    - click
- Rule:
  - Key: [Dictation, pressed]
  - KeyPress:
    - Home
    - click
- Rule:
  - Key: [Left Tilt, pressed]
  - KeyPress:
    - Left
    - click
- Rule:
  - Key: [Right Tilt, pressed]
  - KeyPress:
    - Right
    - click
- Rule:
  - Key: [Forward Button, pressed]
  - KeyPress:
    - XF86_AudioRaiseVolume
    - click
- Rule:
  - Key: [Back Button, pressed]
  - KeyPress:
    - XF86_AudioLowerVolume
    - click
- Rule:
  - Key: [MultiPlatform Gesture Button, pressed]
  - KeyPress:
    - [Super_L, d]
    - click
...
```

```config.yaml
- 1.1.19+202601102018
- _NAME: M720 Triathlon Multi-Device Mouse
  _absent: [hi-res-scroll, lowres-scroll-mode, scroll-ratchet, scroll-ratchet-torque, smart-shift, thumb-scroll-invert, thumb-scroll-mode, onboard_profiles,
    report_rate, report_rate_extended, dpi, dpi_extended, speed-change, backlight, backlight_level, backlight_duration_hands_out, backlight_duration_hands_in,
    backlight_duration_powered, backlight-timed, led_control, led_zone_, rgb_control, rgb_zone_, brightness_control, per-key-lighting, fn-swap, disable-keyboard-keys,
    force-sensing, crown-smooth, divert-crown, divert-gkeys, m-key-leds, mr-key-led, multiplatform, gesture2-gestures, gesture2-divert, gesture2-params,
    haptic-level, haptic-play, sidetone, equalizer, adc_power_management]
  _battery: 4096
  _modelId: B015405E0000
  _sensitive: {divert-keys: true, hires-scroll-mode: ignore, hires-smooth-invert: ignore, hires-smooth-resolution: ignore, persistent-remappable-keys: false,
    reprogrammable-keys: false}
  _serial: 2970F04D
  _unitId: 42E1C161
  _wpid: 405E
  change-host: null
  divert-keys: {82: 0, 83: 1, 86: 1, 91: 1, 93: 1, 208: 1}
  hires-scroll-mode: false
  hires-smooth-invert: false
  hires-smooth-resolution: true
  persistent-remappable-keys: null
  pointer_speed: 256
  reprogrammable-keys: {80: 80, 81: 81, 82: 82, 83: 83, 86: 86, 91: 91, 93: 93, 208: 208}
- _NAME: Logi POP Keys
  _absent: [hi-res-scroll, lowres-scroll-mode, hires-smooth-invert, hires-smooth-resolution, hires-scroll-mode, scroll-ratchet, scroll-ratchet-torque, smart-shift,
    thumb-scroll-invert, thumb-scroll-mode, onboard_profiles, report_rate, report_rate_extended, pointer_speed, dpi, dpi_extended, speed-change, backlight,
    backlight_level, backlight_duration_hands_out, backlight_duration_hands_in, backlight_duration_powered, backlight-timed, led_control, led_zone_, rgb_control,
    rgb_zone_, brightness_control, per-key-lighting, reprogrammable-keys, persistent-remappable-keys, force-sensing, crown-smooth, divert-crown, divert-gkeys,
    m-key-leds, mr-key-led, gesture2-gestures, gesture2-divert, gesture2-params, haptic-level, haptic-play, sidetone, equalizer, adc_power_management]
  _battery: 4100
  _modelId: B36500000000
  _sensitive: {change-host: false, disable-keyboard-keys: false, divert-keys: false, fn-swap: false, hires-scroll-mode: ignore, hires-smooth-invert: ignore,
    hires-smooth-resolution: ignore, multiplatform: false}
  _serial: 4574B50F
  _unitId: 4574B50F
  _wpid: B365
  change-host: null
  disable-keyboard-keys: {1: false, 8: false, 16: false}
  divert-keys: {110: 0, 228: 0, 229: 0, 230: 0, 231: 0, 232: 0, 233: 0, 259: 1, 260: 1, 261: 1, 262: 1, 263: 1, 264: 1, 266: 0, 284: 0}
  fn-swap: false
  multiplatform: 0
- _NAME: Multi Device Silent Mouse M585/M590
  _sensitive: {hires-scroll-mode: ignore, hires-smooth-invert: ignore, hires-smooth-resolution: ignore}
  _serial: D45F50D4
  _wpid: 406B
- _NAME: K850
  _sensitive: {hires-scroll-mode: ignore, hires-smooth-invert: ignore, hires-smooth-resolution: ignore}
  _serial: CAC00B70
  _wpid: '4062'
- _NAME: Logi POP Keys
  _sensitive: {hires-scroll-mode: ignore, hires-smooth-invert: ignore, hires-smooth-resolution: ignore}
  _serial: 4574B50F
  _wpid: B365
- _NAME: Logi POP Mous
  _sensitive: {hires-scroll-mode: ignore, hires-smooth-invert: ignore, hires-smooth-resolution: ignore}
  _serial: C5655072
  _wpid: B030

```


# 除錯相關

## 問題一 每次開機 聲音都變成 線路輸出 可以用一個腳本？ 還是改啥設定 開機預設聲音就預設hdmi輸出嗎？

ai 對策：
這在基於 Ubuntu 的系統（如 Zorin OS）中是很常見的問題。我們可以使用 `pactl` 指令來找出你的 HDMI 音訊設備，並設定為開機預設。以下是最簡單且安全的作法：

### 步驟一：找出你的 HDMI 輸出設備名稱
打開終端機，輸入以下指令：
```bash
pactl list short sinks

```
```bash 結果
winbug3@winbug3-Zorin:~/1T256FREE/Documents/github/zorin_setting_note$ pactl list short sinks
49      alsa_output.pci-0000_0a_00.1.hdmi-stereo        PipeWire        s32le 2ch 48000Hz       SUSPENDED
50      alsa_output.pci-0000_0c_00.4.analog-stereo      PipeWire        s32le 2ch 48000Hz       SUSPENDED
```

這會列出所有的音訊輸出設備。尋找名稱中帶有 `hdmi` 的項目（例如 `alsa_output.pci-0000_01_00.1.hdmi-stereo`），並將該設備名稱複製下來。

### 步驟二：測試切換指令
在終端機輸入以下指令，測試是否能成功切換到 HDMI：
```bash
pactl set-default-sink alsa_output.pci-0000_0a_00.1.hdmi-stereo
```

pactl set-default-sink <剛剛複製的_HDMI_設備名稱>
如果聲音成功改從 HDMI 輸出，表示指令正確。

### 步驟三：設定開機自動執行
我們可以直接透過終端機，在系統的啟動目錄中建立一個自動執行的捷徑檔。請在終端機貼上並執行以下整段指令：

```bash
mkdir -p ~/.config/autostart
cat <<EOF > ~/.config/autostart/set-hdmi-audio-default.desktop
[Desktop Entry]
Type=Application
Exec=pactl set-default-sink alsa_output.pci-0000_0a_00.1.hdmi-stereo
Hidden=false
X-GNOME-Autostart-enabled=true
Name=Set HDMI Audio Default
Comment=開機強制將音訊輸出設為 HDMI
EOF
```
執行完畢後就設定完成了！下次開機時，系統就會自動在背景執行這個指令，將聲音預設為 HDMI 輸出。

### 結果待確認
    2026/5/29 開機確認有效

## 問題二：NVIDIA HDMI 音訊更新斷線修復與持久化方案
這是我之前紀錄的關於系統更新後，導致 NVIDIA HDMI 聲音斷線的修復方案與自動化腳本。

- **Date**: 2026-03-25
- **Environment**: Zorin OS 18 / Linux Kernel 6.17.0 / PipeWire / NVIDIA Proprietary Driver (RTX 3070)

### 🛠 Technical Review
| Status | Method | Key Cause/Error |
| :--- | :--- | :--- |
| ✅ Success | 系統與音訊狀態檢測 (`lspci`, `aplay`, `pactl`) | 確認硬體 (GPU/ALSA) 正常載入，但 PipeWire `Active Profile` 為空，判定為音訊伺服器未成功橋接硬體。 |
| ✅ Success | 清除 WirePlumber 狀態暫存並重啟服務 | 系統更新 (Kernel/DKMS) 導致 ALSA 重新分配 HDMI 硬體 ID。WirePlumber 讀取舊暫存檔造成「ID 錯位」，清除暫存即可強制重新握手恢復音訊。 |
| ✅ Success | 設定 NVIDIA Early KMS (`nvidia-drm.modeset=1`) | 處理時序競爭 (Race Condition)：NVIDIA 驅動載入速度慢於音訊伺服器啟動速度。透過 Initramfs 提早掛載 DRM，確保開機時 HDMI 音軌已備妥。 |
| ✅ Success | 建立 `wireplumber-state-cleaner` 自動清理服務 | 建立免 root 的 Systemd 使用者守護進程，每次開機自動清空狀態檔，一勞永逸解決未來任何更新引發的 ID 錯位問題。 |

### 📦 Dependencies and Configuration
* **音訊伺服器**：PipeWire + WirePlumber
* **顯示卡**：NVIDIA GeForce RTX 3070 / 專有驅動 590.48.01
* **核心模組**：`snd_hda_intel`, `nvidia_drm`
* **系統權限**：`sudo` (修改系統啟動參數) / `systemctl --user` (註冊使用者服務)

### 🚀 Execution Script
```bash
# ==========================================
# Step 1: 立即恢復音訊 (手動清除暫存與重啟服務)
# ==========================================
rm -rf ~/.local/state/wireplumber
systemctl --user daemon-reload
systemctl --user restart wireplumber pipewire pipewire-pulse
sleep 3

# ==========================================
# Step 2: 強制 NVIDIA 驅動提早掛載 (Early KMS)
# 確保開機時 ALSA 模組在音訊伺服器啟動前完全載入
# ==========================================
sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="/GRUB_CMDLINE_LINUX_DEFAULT="nvidia-drm.modeset=1 /g' /etc/default/grub
sudo update-grub

echo -e "nvidia\nnvidia_modeset\nnvidia_uvm\nnvidia_drm" | sudo tee -a /etc/initramfs-tools/modules > /dev/null
sudo update-initramfs -u

# ==========================================
# Step 3: 建立 WirePlumber 狀態自動清理機制
# 徹底根除未來系統更新後硬體 ID 錯位殘留問題
# ==========================================
mkdir -p ~/.config/systemd/user/
cat << 'EOF' > ~/.config/systemd/user/wireplumber-state-cleaner.service
[Unit]
Description=Clean WirePlumber State Before Start
Before=wireplumber.service

[Service]
Type=oneshot
ExecStart=/usr/bin/rm -rf %h/.local/state/wireplumber
RemainAfterExit=yes

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable wireplumber-state-cleaner.service

# ==========================================
# Step 4: 系統重啟以套用所有底層變更
# ==========================================
# sudo reboot
```
