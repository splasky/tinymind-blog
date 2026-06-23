---
title: Qt + Display 造成 Segmentation Fault 的原因與解法
date: 2026-06-23T08:47:51.000Z
---

## 現象

執行 PyQt5 應用程式時加上 `sudo`：

```bash
sudo venv/bin/python3 sparrow-wifi.py
```

直接 **Segmentation Fault (SIGSEGV)**，沒有任何錯誤訊息，或出現：

```
qt.qpa.plugin: Could not load the Qt platform plugin "xcb" in "" even though it was found.
This application failed to start because no Qt platform plugin could be initialized.
```

## 環境

- Ubuntu 24.04 / GNOME Wayland
- Python 3.12 + PyQt5
- ASUS Zenbook UM5302TA

---

## 原因：三個層級的失效

### 1. `sudo` 清空 Display 環境變數

Ubuntu 的 `sudo` 預設啟用 `env_reset`，會把所有使用者環境變數清除，只保留最小集合。被清除的關鍵變數包括：

| 變數 | 用途 |
|---|---|
| `DISPLAY` | X11 / XWayland 顯示編號（預設 `:0`）|
| `WAYLAND_DISPLAY` | Wayland socket 名稱（預設 `wayland-0`）|
| `XAUTHORITY` | X11 認證 cookie 檔案路徑 |
| `XDG_RUNTIME_DIR` | Wayland socket 所在的父目錄 (`/run/user/1000`) |
| `DBUS_SESSION_BUS_ADDRESS` | D-Bus session bus 位址 |

沒有這些變數，Qt5 的 `QApplication()` 無法找到顯示伺服器而直接崩潰。

### 2. `XDG_RUNTIME_DIR` 缺失導致 Wayland plugin 初始化失敗

即使手動設了 `DISPLAY=:0`，如果 `XDG_RUNTIME_DIR` 未設定，Qt5 的 Wayland platform plugin 在嘗試 `wl_display_connect()` 時會因為找不到 socket 路徑而回傳 `NULL`。隨後 Qt5 的 plugin 載入器在迭代可用 plugin 清單時，因為 Wayland plugin 返回了未預期的空指標而觸發 **SIGSEGV**。

測試驗證：
- 有 `XDG_RUNTIME_DIR` → `QApplication()` 成功 ✅
- 無 `XDG_RUNTIME_DIR` → `QApplication()` SIGSEGV ❌

### 3. GNOME Wayland 下 `QT_QPA_PLATFORM` 預設行為導致 XCB 路徑崩潰

GNOME 的 Qt5 整合層有一個特殊邏輯：當偵測到 `XDG_SESSION_TYPE=wayland` 時，Qt5 會**忽略** Wayland 並顯示以下警告：

```
Warning: Ignoring XDG_SESSION_TYPE=wayland on Gnome. Use QT_QPA_PLATFORM=wayland to run on Wayland anyway.
```

接著 Qt5 改載入 `xcb` (XWayland) plugin。但在 root 權限下，XWayland 的連線同樣會因權限問題失敗而 segfault。

必須**明確設定** `QT_QPA_PLATFORM=wayland` 才能強制使用 Wayland plugin，而這個 plugin 在 GNOME Mutter 下允許 root 連線（無 `SO_PEERCRED` 阻擋）。

---

## 解法

在 `QApplication(sys.argv)` 之前，偵測是否為 root 執行，並從原使用者的 session 還原關鍵環境變數：

```python
if os.geteuid() == 0:
    orig_uid = os.environ.get('SUDO_UID',
               os.environ.get('PKEXEC_UID', ''))
    orig_user = os.environ.get('SUDO_USER', '')
    if not orig_uid and not orig_user:
        for entry in os.listdir('/run/user/'):
            if entry.isdigit():
                uid_path = f'/run/user/{entry}'
                if os.path.isdir(uid_path) and os.access(uid_path, os.R_OK):
                    wl_test = os.path.join(uid_path, 'wayland-0')
                    if os.path.exists(wl_test):
                        orig_uid = entry
                        break

    if orig_uid and 'XDG_RUNTIME_DIR' not in os.environ:
        xdg_rt = f'/run/user/{orig_uid}'
        if os.path.isdir(xdg_rt):
            os.environ['XDG_RUNTIME_DIR'] = xdg_rt

    if 'DISPLAY' not in os.environ:
        os.environ['DISPLAY'] = ':0'

    if orig_uid and 'WAYLAND_DISPLAY' not in os.environ:
        wl_socket = f'/run/user/{orig_uid}/wayland-0'
        if os.path.exists(wl_socket):
            os.environ['WAYLAND_DISPLAY'] = 'wayland-0'

    if 'XAUTHORITY' not in os.environ and orig_uid:
        xauth_dir = f'/run/user/{orig_uid}'
        if os.path.isdir(xauth_dir):
            for entry in os.listdir(xauth_dir):
                if entry.startswith('.mutter-Xwaylandauth'):
                    os.environ['XAUTHORITY'] = os.path.join(xauth_dir, entry)
                    break

    if 'DBUS_SESSION_BUS_ADDRESS' not in os.environ and orig_uid:
        bus_path = f'/run/user/{orig_uid}/bus'
        if os.path.exists(bus_path):
            os.environ['DBUS_SESSION_BUS_ADDRESS'] = f'unix:path={bus_path}'

    if 'WAYLAND_DISPLAY' in os.environ and 'QT_QPA_PLATFORM' not in os.environ:
        os.environ['QT_QPA_PLATFORM'] = 'wayland'
```

---

## 解法總結

| 步驟 | 變數 | 原因 |
|---|---|---|
| 1 | `XDG_RUNTIME_DIR` | Wayland socket 位置，缺失則 Wayland plugin 空指標崩潰 |
| 2 | `DISPLAY` | XWayland 顯示編號 |
| 3 | `WAYLAND_DISPLAY` | Wayland socket 名稱 |
| 4 | `XAUTHORITY` | XWayland 認證 cookie |
| 5 | `DBUS_SESSION_BUS_ADDRESS` | D-Bus 整合 |
| 6 | `QT_QPA_PLATFORM=wayland` | ★ 防止 GNOME 自動降級到 XCB（XCB 在 root 下會 segfault）|

---

## 附：快速測試指令

```bash
pkexec env DISPLAY=:0 XDG_RUNTIME_DIR=/run/user/1000 \
  WAYLAND_DISPLAY=wayland-0 \
  XAUTHORITY=/run/user/1000/.mutter-Xwaylandauth.* \
  QT_QPA_PLATFORM=wayland \
  python3 -c "
from PyQt5.QtWidgets import QApplication
import sys
app = QApplication(sys.argv)
print('QApp OK')
app.quit()
"
```

預期輸出：
```
QStandardPaths: runtime directory '/run/user/1000' is not owned by UID 0, ...
QApp OK
```

`QStandardPaths` 的警告是良性的（不影響功能），看到 `QApp OK` 表示修復成功。
