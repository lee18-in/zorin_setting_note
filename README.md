# Zorin OS Setting Notes
My Zorin OS configuration notes.

這個github專案是私人倉庫 用於記錄 Zorin OS 的各項設定筆記。我期望透過與 AI 協作，探索並實現「以自然語言修改系統設定」的目標。

# 設定相關

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

