# Shell Startup 除錯案例

## 案例一：`.bashrc` 孤兒 `fi`

**症狀：** SSH 登入時 bash 報 syntax error，或無感但 `bash -n` 抓到。

**原始程式碼（第 146-154 行）：**

```bash
# ── tmux — SSH 斷線不中斷 ──
if [[ $- == *i* ]] && command -v tmux &>/dev/null; then
  if ! tmux has-session -t work 2>/dev/null; then
    tmux new-session -ds work
  fi
  # if [ -n "$SSH_CONNECTION" ] && [ -z "$TMUX" ]; then
  #   tmux attach -t work
  fi           # ← 孤兒 fi！原本屬於被註解掉的 if
fi
```

**根因：** 原本有三層 if 嵌套。中間那層 `if [ -n "$SSH_CONNECTION" ]`（含其 body 和 fi）被註解掉時，**fi 忘了跟著註解**，導致多出一個無主的 `fi`。

**修復：** 刪除 153 行的孤兒 fi

```bash
# 確認行號
sed -n '153p' ~/.bashrc   # 應只輸出 "fi"

# 備份 + 刪除
cp ~/.bashrc ~/.bashrc.bak
sed -i '153d' ~/.bashrc

# 驗證
bash -n ~/.bashrc && echo "✅ 語法通過"
```

---

## 案例二：fzf `--bash` 不相容

**症狀：** SSH 登入時報 `unknown option: --bash`

**環境：** Ubuntu 24.04, fzf 0.44.1 (Debian 套件)

**根因：** fzf 0.44.1 不支援 `--bash` 參數（v0.48+ 才加入）。Debian/Ubuntu 套件的 fzf 通常落後數個版本。

**檢查：**

```bash
fzf --version          # → 0.44.1 (debian)
fzf --bash 2>&1        # → unknown option: --bash
```

**修復：**

```bash
# 確認 key-bindings 腳本存在
ls /usr/share/doc/fzf/examples/key-bindings.bash

# 取代 eval "$(fzf --bash)" 行
sed -i 's|eval "$(fzf --bash)" 2>/dev/null|source /usr/share/doc/fzf/examples/key-bindings.bash 2>/dev/null|' ~/.bashrc
```

**若需要 `--bash` 支援（安裝新版）：**

```bash
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install
# 新版會使用 eval "$(fzf --bash)" 語法
```

注意新版 fzf 安裝路徑不同，key-bindings 腳本在 `~/.fzf/shell/key-bindings.bash`。

---

## 案例三：thefuck Python 3.12 不相容

**症狀：** SSH 登入時 `thefuck --alias` 噴 traceback（若無 `2>/dev/null` 則明顯可見）

**環境：** Ubuntu 24.04, Python 3.12, thefuck 3.29 (apt)

**根因：** thefuck 使用了 Python 3.12 已完全移除的 `imp` 模組。

**檢查：**

```bash
thefuck --alias 2>&1 | head -3
# Traceback (most recent call last):
#   File "/usr/bin/thefuck", line 9, in <module>
#     from thefuck.entrypoints.main import main
#   File "/usr/share/thefuck/thefuck/entrypoints/main.py", line 8, in <module>
#     from .. import logs  # noqa: E402
```

**修復：**

```bash
# 移除套件（不常用則不值得升級）
sudo apt remove thefuck
sudo apt autoremove          # 清殘留相依套件

# 移除 .bashrc 中的 eval 行（否則 command not found）
sed -i '/thefuck/d' ~/.bashrc
```

**升級路線（若需要保留）：**

```bash
pip install --upgrade thefuck --break-system-packages
# 但 pip 版可能也已不再維護。建議換 modern 替代品。
```
