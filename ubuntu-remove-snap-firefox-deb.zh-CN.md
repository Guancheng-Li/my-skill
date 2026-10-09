---
name: ubuntu-remove-snap-firefox-deb
description: 卸载 Ubuntu 上的 snap（含 snapd）并用 Mozilla 官方 APT 源安装 deb 版 Firefox。当用户要求"卸载 snap / 彻底删除 snapd / 装 Firefox 但不要 snap 版 / 换成 deb 版 Firefox"时使用。
---

# Ubuntu 彻底卸载 snap + 安装 deb 版 Firefox

按顺序执行两大阶段：**先卸 snap，后装 Firefox**。所有 `sudo` 步骤需要用户密码——把命令交给用户执行，不要假装能自己跑。全程不引入任何 snap 组件。

## 阶段一：卸载 snap

### 步骤 1：摸底

```bash
snap list                          # 已装的 snap 包
dpkg -l | grep -iE "firefox|snapd" # firefox 是否为 snap 过渡包（版本形如 1:1snap1-0ubuntu2）
apt-cache rdepends --installed snapd
apt-get -s purge snapd             # 模拟：确认 purge 实际会删哪些包
```

完成判据：拿到 snap 包清单，且模拟输出中**没有**会连带卸载桌面组件（gnome-shell、pulseaudio 等）的迹象，才进入步骤 2。

关键事实：`ubuntu-desktop` / `ubuntu-desktop-minimal` 对 snapd 只是 **Recommends（推荐）**，不是 Depends。`apt-get -s purge snapd` 应只显示删除 `snapd` 和 `firefox` 两个包——这是继续的安全信号。若模拟显示会删更多，停下来分析。

### 步骤 2：逐个移除 snap 包（应用 → 运行时 → base → snapd 自身）

```bash
sudo snap remove --purge <应用包>            # firefox、snap-store 等真实应用
sudo snap remove --purge <运行时包>          # gnome-XX-XXXX、gtk-common-themes、mesa-XXXX
sudo snap remove --purge bare core20 core22 core24
sudo snap remove --purge snapd
```

按实际清单替换包名，先删应用再删 base。base 被占用拒绝删除时**跳过即可**，下一步会强制处理。

### 步骤 3：删除系统包与残留

```bash
sudo apt purge -y snapd firefox              # firefox 此处指 snap 过渡包
sudo apt autoremove --purge -y
sudo rm -rf /var/lib/snapd /var/snap /snap ~/snap
hash -r
```

有 squashfs 挂载点删不掉时，重启后重跑 `rm -rf`。

### 步骤 4：验证

```bash
which snap snapd          # 均为 command not found
systemctl status snapd    # unit could not be found
ls -d /var/lib/snapd /var/snap /snap ~/snap   # 全部不存在
mount | grep snap         # 无 squashfs 挂载
```

四项全过即阶段一完成。

### 卸载陷阱（保留 vs 删除）

| 包 | 处置 | 原因 |
|---|---|---|
| `libsnappy1v5` | **保留** | 名字像 snap 实为压缩库，Chrome 等依赖 |
| `xdg-desktop-portal` | **保留** | GTK 文件对话框等桌面功能依赖，描述里提 snap 不代表依赖 snapd |
| `libsnapd-glib1` / `gir1.2-snapd-1` | **保留** | 删除会连带卸载 PulseAudio、GNOME 在线账户；无 snapd 后闲置无害 |
| `ubuntu-desktop` 元包 | 保留（自动） | 只被 Recommends 关联，purge snapd 不会动它 |

## 阶段二：Mozilla 官方 APT 源安装 Firefox（deb 版）

### 步骤 5：测源可达性

```bash
curl -sI --max-time 8 https://packages.mozilla.org/apt/dists/mozilla/Release
```

预期 `HTTP/2 200`。被墙时改走本地代理再测。不通则中止并报告。

### 步骤 6：配源 + 双保险 pin

```bash
sudo install -d -m 0755 /etc/apt/keyrings
wget -q https://packages.mozilla.org/apt/repo-signing-key.gpg -O- | \
  sudo tee /etc/apt/keyrings/packages.mozilla.org.asc > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/packages.mozilla.org.asc] https://packages.mozilla.org/apt mozilla main" | \
  sudo tee /etc/apt/sources.list.d/mozilla.list

sudo tee /etc/apt/preferences.d/mozilla <<'EOF'
Package: *
Pin: origin packages.mozilla.org
Pin-Priority: 1000

Package: firefox
Pin: release o=Ubuntu*
Pin-Priority: -1
EOF
```

pin 的两层含义：Mozilla 源优先级 1000 保证永远压过 Ubuntu 源；Ubuntu 源的 firefox 直接打到 -1（永久拒绝），snap 过渡包（`1:1snap1-*`）无法回归。

### 步骤 7：安装并验证

```bash
sudo apt update && sudo apt install -y firefox firefox-l10n-zh-cn
apt policy firefox
```

完成判据（`apt policy firefox` 输出）：
- 已安装版本形如 `157.0.1~build1`（`~build1` 后缀 = Mozilla deb）
- 安装源为 `https://packages.mozilla.org/apt`，优先级 1000
- Ubuntu 源的 `1:1snap1-*` 版本标记为 `-1`

三项全中即整个任务完成。此后 `apt upgrade` 自动跟进 Mozilla 的新版本，全程与 snapd 无关。
