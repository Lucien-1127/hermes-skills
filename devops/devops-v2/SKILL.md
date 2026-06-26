---
name: devops-v2
description: 系統監控、效能優化、自動維護 — 三層閾值 (CPU/Memory/Disk) + 清理管線
version: 2.4.0
author: Lucian
trigger: "輸入包含「CPU」「記憶體」「磁碟」「監控」「優化」「效能」「bashrc」「shell」「啟動錯誤」「fzf」，或 Curator 定時觸發維護"
---

# DevOps Skill v2.0

## 用途
系統監控、當機診斷、效能優化、自動維護任務。

## 監控指標（三層閾值）

| 指標 | 正常 | 警告 (🟡) | 危急 (🔴) | 行動 |
|------|------|---------|---------|------|
| CPU | <70% | 70–85% | >85% | SKIP high-load task |
| Memory | <80% | 80–90% | >90% | Kill non-essential / Notify |
| Disk | <80% | 80–90% | >90% | Cleanup / Archive |
| Swap | 有配置 | 無 swap | — | 建立 /swapfile（RAM 的 25-50%） |

## 當機診斷管線（由淺入深）

當使用者回報 VM 不穩定/常當機，依序執行以下診斷：

### Phase 1 — 快照指標（8 指令群）
```bash
uptime                                    # 運行時間 & load average
who -b; last -x -F reboot shutdown | head -20  # 開機歷史
free -h; swapon --show                    # 記憶體 & swap
df -h --exclude-type=tmpfs --exclude-type=devtmpfs | sort -k5 -h -r  # 磁碟（濾掉 tmpfs）
df -i --exclude-type=tmpfs --exclude-type=devtmpfs | head -10        # inode
ps aux --sort=-%cpu | head -10            # 最吃 CPU 的 process
ps aux --sort=-%mem | head -10            # 最吃記憶體的 process
ss -tlnp 2>/dev/null | head -20           # 監聽埠
who                                       # 目前 SSH 連線
```

### Phase 2 — kernel & 硬體層（3 指令）
```bash
sudo dmesg --level=emerg,alert,crit,err,warn | tail -60   # kernel 錯誤
sudo sgdisk -v /dev/nvme0n1 2>/dev/null                   # GPT 完整性（若 NVMe）
sudo fdisk -l /dev/nvme0n1 2>/dev/null | grep "Disk"      # 磁碟幾何 & GPT type
```

常見 kernel 錯誤訊號：
- `Alternate GPT header not at the end of the disk` → GPT 備份標頭損毀，`sgdisk -e` 修復
- `NVRM: No NVIDIA GPU found` → 無 GPU 機器裝了 NVIDIA 驅動，移除
- `Out of memory` / `oom_kill` → 記憶體不足，檢查 leak 或加 swap
- `GPT:Primary header thinks Alt. header is not at the end` → 同上 GPT 問題

**常見無害訊息（不需處理）：**
- `martian source <IP> from <IP>, on dev tailscale0` → Tailscale 健康檢查封包，正常行為
- `kauditd_printk_skb: X callbacks suppressed` → auditd 被 rate-limit，大量連線的正常現象
- `Found left-over process <PID> (<name>) in control group while starting unit` → systemd 發現 tmux/syncthing/hermes 等使用者程序在前次 SSH 結束後仍在跑，預期行為，無需處理

### Phase 3 — systemd & 服務層（5 指令）
```bash
sudo systemctl --failed                                    # 失敗服務
sudo journalctl -p 0..3 --no-pager -n 50                   # 嚴重/錯誤日誌
sudo journalctl -g "oom_kill|OOM|Out of memory|killed process" -n 30  # OOM
# 背景服務檢查（無頭伺服器常見）
ps aux | grep -E "syncthing|tailscaled|hermes.*gateway" | grep -v grep
tailscale status 2>/dev/null                               # Tailnet 節點狀態
```

### Phase 4 — 日誌 & crash dump
```bash
ls -la /var/crash/          # apport crash reports（移除套件後檢查殘留）
sudo journalctl -u sshd --no-pager -n 20   # SSH 錯誤（暴力攻擊）
sudo journalctl --vacuum-time=3d           # 壓縮舊日誌
cat /var/log/kern.log | grep -i "nvidia\\|NVRM\\|panic\\|oom"  # kernel log 專屬檢查
cat /var/lib/gce-accelerator/shipped-nvidia-version 2>/dev/null  # GCE GPU flag 殘留
journalctl --disk-usage                    # 日誌磁碟用量
```

## Phase 5 — 可優化項目掃描（例行健檢補充）

健康檢查或系統優化時加跑：

```bash
# 快取用量
du -sh ~/.cache/ /var/cache/apt/ /var/log/journal/
# 可升級套件
apt list --upgradable 2>/dev/null | grep -v "Listing..."
# 舊 kernel（比對目前版本）
dpkg -l linux-image-* 2>/dev/null | grep ^ii | awk '{print $2}' | grep -v "$(uname -r)"
# NVMe 硬碟
sudo nvme list 2>/dev/null
# Hermes gateway 數量
ps aux | grep 'hermes.*gateway' | grep -v grep
```

## 修復行動檢查清單

發現問題後，按優先級逐項修復：

1. **GPT 分割表損毀** → `sudo sgdisk -e /dev/nvme0n1`（重建備份 GPT header）
2. **無 swap** → `fallocate -l 2G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile`，寫入 `/etc/fstab`
3. **多餘驅動**（NVIDIA 等無 GPU 機器） → 完整清理序列：
   ```bash
   # 步驟 1: purge 所有 nvidia 套件
   sudo apt-get purge -y nvidia-* linux-modules-nvidia-* libnvidia-*
   # 步驟 2: 清除 GCE accelerator 旗標（防下次開機自動載入）
   echo "" | sudo tee /var/lib/gce-accelerator/shipped-nvidia-version
   # 步驟 3: 加入 modprobe 黑名單（封鎖 nvidia/nvidia_drm/nvidia_modeset/nvidia_uvm）
   sudo tee /etc/modprobe.d/blacklist-nvidia.conf <<'EOF'
   blacklist nvidia
   blacklist nvidia_drm
   blacklist nvidia_modeset
   blacklist nvidia_uvm
   EOF
   # 步驟 4: 刪除殘留 systemd unit
   sudo rm -f /etc/systemd/system/nvidia-fabricmanager.service \
             /etc/systemd/system/nvidia-persistenced.service
   # 步驟 5: 重建 initramfs（purge 不一定會自動做）
   sudo update-initramfs -u -k all
   # 步驟 6: 刪除舊 kernel 的 initramfs（可能含 nvidia 殘留設定）
   ls /boot/initrd.img-*generic* 2>/dev/null | while read f; do
     [ -f "$f" ] && lsinitramfs "$f" | grep -q nvidia && sudo rm -v "$f"
   done
   # 步驟 7: 清理 systemd 記憶的失敗狀態
   sudo systemctl reset-failed nvidia-fabricmanager.service 2>/dev/null
   sudo systemctl reset-failed nvidia-persistenced.service 2>/dev/null
   sudo systemctl daemon-reload
   ```
   ⚠️ **常見陷阱**：purge 後 initramfs 不一定會自動重建。務必手動執行 `update-initramfs -u -k all`，且檢查所有 kernel 版本的 initramfs 是否有 nvidia 殘留。GCE VM 的 `/var/lib/gce-accelerator/shipped-nvidia-version` 旗標若未清除，下次開機仍會嘗試載入 nvidia。
4. **OOM 風險** → 確認 swap 已建立，檢查是否有 leak（`ps aux --sort=-%mem` 高駐留程序）
5. **系統過期** → `apt upgrade -y`，有 kernel 更新需提示重啟
6. **GPT partition table entries not in disk order** → 通常是 cloud-init image 的正常現象，不處理
7. **套件移除後 crash report 殘留** → `sudo rm /var/crash/*.crash 2>/dev/null`，移除已不存在的套件的 crash dump

## Shell Startup 檔案除錯

`.bashrc`、`.profile`、`.bash_profile` 等登入腳本出錯時，依序處理：

### 典型錯誤模式

**1. 孤兒 `fi` / `if`（最常見）**

註解掉 if 區塊內文時，忘了也註解掉原本的 `fi`：

```bash
if condition; then
  # if inner_condition; then    ← 被註解
  #   do_something               ← 被註解
  fi                             ← ← 孤兒 fi！沒 if 可關
fi
```

**診斷方法：**
```bash
bash -n ~/.bashrc                        # 語法檢查
# 找到出錯行號後，讀取前後 15 行確認結構
sed -n '$((LINE-10)),$((LINE+5))p' ~/.bashrc
```

**修復：** 刪除多餘的控制字，或補回被註解掉的 if/fi。備份優先：
```bash
cp ~/.bashrc ~/.bashrc.bak && sed -i 'LINE_NUMd' ~/.bashrc
```

**2. `eval "$(cmd --flag)"` 不相容**

工具版本不支援特定 flag 時，eval 會產出錯誤輸出：

```bash
# 錯誤範例（fzf 0.44.1 不支援 --bash）
eval "$(fzf --bash)" 2>/dev/null        # → unknown option: --bash

# 修正：改用正確的整合方式
source /usr/share/doc/fzf/examples/key-bindings.bash 2>/dev/null
```

**檢查流程：**
```bash
cmd --version                           # 確認版本
cmd --flag 2>&1 | head -3               # 測試 flag 是否存在
type -a cmd                             # 確認來源（apt vs 手動安裝 vs pip）
apt-cache policy cmd                    # 看 apt 版本
```

⚠️ **陷阱：** `2>/dev/null` 會隱藏錯誤，登入時無感但 `.bashrc` 確實有問題。修復時要連同錯誤訊息一起看，不能只相信 exit code。

### 完整檢查流程

```bash
# 1. 語法檢查
bash -n ~/.bashrc ~/.profile 2>&1

# 2. 找出所有 eval 行（高風險來源）
grep -n "eval" ~/.bashrc

# 3. 逐行測試
source ~/.bashrc 2>&1                  # 看實際報錯

# 4. 確認無殘留錯誤後重登入驗證
```

三種測試：逐行測試 `source ~/.bashrc`。最終驗證：登出重登入後觀察有無報錯。

完整案例與錯誤輸出參考 `references/shell-startup-troubleshooting.md`。

## 套件版本相容性（Ubuntu 24.04 常見問題）

```bash
lsb_release -a                          # 確認 Ubuntu 版本
python3 --version                       # 24.04 預設 Python 3.12
```

### Python 3.12 斷相容清單

| 套件 | 問題 | 解法 |
|------|------|------|
| **thefuck** | `imp` 模組被移除，import 即炸 | `sudo apt remove thefuck`（不常用則不值得修）|
| 其他舊 pip 套件 | 依賴已移除的 stdlib 模組 | `pip install --upgrade` 或換替代方案 |

**判斷原則：** 若該套件在 `eval "$(cmd --alias)"` 形式被引用，加上 `2>/dev/null` 可隱藏錯誤但不治本。不常用的直接移除，常用的找現代替代品。

### fzf 版本陷阱

Ubuntu 24.04 套件庫提供 fzf 0.44.1，不支援 `--bash` flag（需 0.48+）：

```bash
# ❌ 不能用（0.44.1 無效）
eval "$(fzf --bash)" 2>/dev/null

# ✅ 正確（0.44.x 適用）
source /usr/share/doc/fzf/examples/key-bindings.bash 2>/dev/null

# 若需 --bash 支援，從 GitHub 安裝新版
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install
```

⚠️ 安裝新版前先確認 `/usr/share/doc/fzf/examples/key-bindings.bash` 是否存在——Debian 套件的 key-bindings 腳本位置與 GitHub 版不同。

## 自動化清理清單

1. **暫存檔** — ~/.cache/, /tmp/ (>7 日)
2. **舊日誌** — /var/log/ (>30 日)，journalctl --vacuum-size=100M 或 --vacuum-time=3d
3. **孤立進程** — Zombie / Defunct 進程清理（`ps aux | awk '$8 ~ /Z/ {print $2}' | xargs kill`）
4. **套件快取** — `apt-get clean` 清理 /var/cache/apt/，`pip cache purge`，`uv cache clean`
5. **crash reports** — /var/crash/ 內的非核心 crash 可清除
6. **套件更新** — 檢查更新列表，提示使用者

## 執行步驟
1. 收到系統問題回報 → 先確認「當機」還是「慢」
2. 慢 → 直接 Phase 1 + cleanup
3. 當機/不穩定 → 完整 Phase 1→2→3→4
4. 依修復檢查清單執行修復
5. 最終驗證（不可跳過，確認無錯誤才可停止）：
   ```bash
   # 系統基本
   uptime
   free -h
   swapon --show
   df -h /
   sudo systemctl --failed
   # NVIDIA 特定驗證
   modprobe --dry-run nvidia 2>&1                   # FATAL=已封鎖
   cat /etc/modprobe.d/blacklist-nvidia.conf 2>/dev/null   # 確認黑名單存在
   cat /var/lib/gce-accelerator/shipped-nvidia-version     # 應為空
   ls /etc/systemd/system/nvidia* 2>/dev/null || echo "(無殘留)"
   lsinitramfs /boot/initrd.img-$(uname -r) | grep nvidia || echo "(initramfs 乾淨)"
   # 記憶體/磁碟
   ps aux --sort=-%mem | head -5
   ```
6. 若任何驗證項目失敗 → 回到步驟 4 補修復，**必須確認 10 項全過才能回報完成**

## 配置備份安全 — 機敏資料防護

當將系統配置（config.yaml、.env、hermes-config 等）備份或記錄到檔案時：

- ❌ **禁止**在備份檔案中保留未遮罩的 API key、token、password
- ✅ 一律將 `api_key: sk-or-...` 之類替換為 `api_key: <REDACTED>`
- ✅ 系統快照（system.txt、packages.txt）部分不含金鑰，安全
- ✅ 僅記錄**配置結構**，不記錄原始憑證值
- ⚠️ GitHub Push Protection 會自動阻擋含 secrets 的推送，需先清理再 commit

**清理指令：**
```bash
sed -i 's|api_key: sk-or-[^ ]*|api_key: <REDACTED>|g' vm/hermes-config.txt
git add -A && git commit --amend --no-edit && git push --force
```

完整流程與檢查清單參考 `references/config-backup-secrets.md`。

## 安全封鎖時的行為規則

當 agent 執行重啟/關機/停止等系統操作被安全機制封鎖時，**不要嘗試替代方案**。立即輸出以下三項資訊給使用者：

```bash
# 1. 確切指令（使用者可直接複製貼上）
sudo reboot
# 或
sudo systemctl reboot

# 2. 替代路徑（若 SSH 也無法重啟，可能是雲端 VM 加密/鎖定狀態）
GCP 主控台 → Compute Engine → VM 執行個體 → 重設 (Reset)

# 3. 重啟後驗證指令
uname -r    # 確認核心已切換
```

**規則：** 被封鎖後最多嘗試一次替代方案（如 `systemctl reboot`），失敗就直接給使用者指令。繞圈嘗試只會讓等待更久、Token 浪費更多。

## 禁止事項
- ❌ 在 CPU >85% 時執行清理（避免雪崩）
- ❌ 刪除關鍵系統檔案（whitelist only）
- ❌ 無詢問殺死關鍵進程
- ❌ 修改 cloud-init 產生的分割表配置（GPT partition order warnings 可忽略）
- ❌ 無確認就重啟 VM（kernel 更新需使用者同意）
- ❌ 假設 GCP VM 可以透過 SSH 重啟 — 加密/鎖定狀態下的 VM 可能無法從終端機重啟，需透過 GCP 主控台操作
- ❌ 修復後跳過最終驗證就回報完成 — 必須確認所有檢查項目全過（failed services=0, initramfs 無殘留, 模組不可載入等）

## 背景服務管理（無頭伺服器）

在 GCP/Linux 無頭伺服器上，`systemctl --user` 經常失敗：
```
Failed to connect to bus: No medium found
```

這是因為無頭環境沒有 user systemd bus。**解法：直接用 background process 啟動。**

```bash
# ❌ 這會失敗（無 systemd bus）
systemctl --user enable syncthing
systemctl --user start syncthing

# ✅ 用 terminal background flag 啟動
terminal(command="syncthing --no-browser", background=true)

# 管理：process(action="poll"), process(action="log"), process(action="kill")
```

啟動後記得驗證程序確實運行：
```bash
ps aux | grep syncthing | grep -v grep
```

## 參考文件
- `references/gcp-vm-diagnostics.md` — GCP 雲端 VM 專屬檢查項目（含 NVIDIA、GPT、Swap、雲端層操作問題）
- `references/syncthing-headless-setup.md` — Syncthing 無頭伺服器安裝、設定、故障排除
- `references/config-backup-secrets.md` — 配置備份機敏資料清理流程
- `references/shell-startup-troubleshooting.md` — Shell startup 檔案除錯案例（孤兒 fi、fzf 不相容、Python 3.12 斷相容）

## SSH 連線穩定性（行動裝置適用）

行動裝置（iPhone/Android）的 SSH client 在切換 app 或鎖定螢幕時會斷線。以下方案可確保工作中斷後能接回：

### 方案 A：tmux（推薦，最簡單）
```bash
sudo apt-get install -y tmux    # 安裝
tmux new -s work                # 建立持久 session
# 斷線後重新 SSH 連上，執行：
tmux attach -t work             # 接回原本 session，所有工作完整保留
# 其他操作
tmux ls                         # 列出所有 session
tmux detach                     # 脫離 session (Ctrl+B, D)
tmux kill-session -t work       # 刪除 session
```

### 方案 B：mosh（專為行動裝置設計）
```bash
sudo apt-get install -y mosh
# iPhone 需支援 mosh 的 client（如 Blink Shell）
mosh user@host
```

### 方案 C：SSH KeepAlive（客戶端設定）
在 `~/.ssh/config` 加入：
```
Host *
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

### 注意事項
- tmux 不需額外 client 支援（標準 SSH 即可用）
- mosh 需要 UDP（部分防火牆會封鎖）
- 同時開多個 session 用 `tmux ls` + `tmux attach -t <name>` 管理

## 成本與參數
- 溫度：0.1（確定性最高）
- 最大輸出 Token：512（診斷管線較長）
- 信心閾值：85%
- 成本上限：$0.15/次
- 延遲目標：<10s