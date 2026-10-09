# Ubuntu 22.04 编译安装 Ghostty 完整流程

## 概述

在 Ubuntu 22.04（jammy，glibc 2.35）上从源码编译安装 Ghostty 终端模拟器。

参考项目：`~/github/ghostty-ubuntu22.04`（fork 自 [Maix0/ghostty-ubuntu22.04](https://github.com/Maix0/ghostty-ubuntu22.04)）

### 核心难点

Ghostty 官方只提供 macOS 和 Linux（glibc 2.38+ / Ubuntu 24.04）版本。22.04 的 glibc 是 2.35，缺少 `GLIBC_2.38` 的符号版本。本流程通过**按 glibc 2.35 目标交叉编译 + 捆绑 GTK 库 + 符号 shim** 三步解决。

## 环境要求

| 项目 | 要求 | 说明 |
|------|------|------|
| 系统 | Ubuntu 22.04.5 LTS | glibc 2.35 |
| Docker | 免 sudo 可用 | `docker ps` 需正常 |
| 磁盘 | ~5 GB | 依赖缓存约 500MB，容器约 1.8GB |
| 网络 | 能访问 github.com / alpine CDN | 见「已知陷阱」 |

## 阶段一：准备构建环境

### 1. 固定 zig 版本

Ghostty 1.1.3 的 `build.zig.zon` 用的是 zig 0.13 时代的 schema。zig 0.14+ 改了 `.name` 字段的解析规则，会报 `expected enum literal`。**必须用 0.13.0**。

```bash
mkdir -p ~/opt/zig013
cd ~/opt/zig013
curl -L 'https://ziglang.org/download/0.13.0/zig-linux-x86_64-0.13.0.tar.xz' | tar -xJ
# 验证
./zig-linux-x86_64-0.13.0/zig version   # 应输出 0.13.0
```

### 2. 预取构建依赖（关键）

**zig 内置的 TLS 栈不读系统 CA 证书包**，直接 `zig build` 拉依赖会报 `TlsInitializationFailed`，而同一时刻 `curl` 完全正常。

解法：用 curl 下载所有依赖 tarball，再逐个 `zig fetch` 喂进全局缓存。

```bash
# 下载 ghostty 源码
mkdir -p /tmp/ghostty-src && cd /tmp/ghostty-src
curl -L 'https://github.com/ghostty-org/ghostty/archive/refs/tags/v1.1.3.tar.gz' \
    | tar -xz --strip-components=1

# 提取所有依赖 URL，用 curl 下载后喂给 zig 缓存
python3 - <<'PY'
import re, subprocess, pathlib, time
ZIG = str(pathlib.Path.home() / "opt/zig013/zig-linux-x86_64-0.13.0/zig")
files = ["/tmp/ghostty-src/build.zig.zon"] + \
        [str(p) for p in pathlib.Path("/tmp/ghostty-src/pkg").glob("*/build.zig.zon")]
urls = set()
for f in files:
    try: urls |= set(re.findall(r'https://[^"]+\.tar\.gz', pathlib.Path(f).read_text()))
    except OSError: pass
urls = sorted(urls)
print(f"共 {len(urls)} 个依赖")
ok = fail = 0
for u in urls:
    dest = "dep.tar.gz"
    for _ in range(4):   # 网络抖动重试
        r = subprocess.run(["/usr/bin/curl","-sL","--max-time","180","-o",dest,u],
                           capture_output=True)
        p = pathlib.Path(dest)
        if r.returncode == 0 and p.exists() and p.stat().st_size > 0: break
        time.sleep(2)
    else:
        print(f"DL-FAIL {u}"); fail += 1; continue
    if subprocess.run([ZIG,"fetch",dest], capture_output=True).returncode == 0: ok += 1
    else: print(f"FETCH-FAIL {u}"); fail += 1
print(f"成功 {ok}, 失败 {fail} / {len(urls)}")
PY

# 复制一份给容器用
rm -rf ~/.cache/zig-persist
cp -a ~/.cache/zig ~/.cache/zig-persist
du -sh ~/.cache/zig-persist   # 约 517MB
```

## 阶段二：容器内编译

### 构建脚本

保存为 `/tmp/build-ghostty.sh`：

```bash
#!/bin/sh
set -e

# 旧容器留下的索引缓存会导致 apk update 报 Permission denied
rm -rf /var/cache/apk/* 2>/dev/null || true
apk update -q >/dev/null 2>&1 || true

# alpine 仓库偶发 TLS 失败，重试直到装齐
for i in 1 2 3 4 5; do
  if apk add --no-cache \
    xz ncurses glib glib-dev libadwaita-dev \
    harfbuzz-static harfbuzz-dev pango-dev gdk-pixbuf-dev \
    gtk4.0 gtk4.0-dev wayland wayland-dev wayland-protocols \
    libx11-dev pkgconf fribidi-dev util-linux-dev >/dev/null 2>&1; then
    break
  fi
  echo "apk add 第 $i 次失败，重试..."
  sleep 3
done
# apk add 部分失败时仍返回 0，必须显式验证
apk info -e gtk4.0-dev libx11-dev wayland-dev fribidi-dev >/dev/null 2>&1 \
  || { echo "关键包未装齐"; exit 1; }

# ── 修复 pkg-config 的 .pc 文件 ──
# alpine 的 x11.pc / wayland-*.pc 写 Cflags: -I${includedir}，pkgconf 展开后为空；
# 而展开成 -I/usr/include 又会让 zig 优先读到 musl 头（sys/cdefs.h），
# 与 zig 自带的 glibc 头冲突。折中：把 X11/wayland 头复制到独立目录。
mkdir -p /opt/inc
cp -r /usr/include/X11 /opt/inc/
cp /usr/include/wayland*.h /opt/inc/ 2>/dev/null || true
cp -r /usr/share/wayland-protocols /opt/inc/ 2>/dev/null || true

emit_pc() {  # name, description, version, requires, libs
  cat > "/usr/lib/pkgconfig/$1.pc" <<PC
prefix=/usr
Name: $2
Description: $2
Version: $3
Requires: ${4:-}
Cflags: -I/opt/inc
Libs: -L/usr/lib $5
PC
}
emit_pc x11 "X Library" 1.8.9 "xproto kbproto" "-lX11 -lpthread"
emit_pc wayland-client "Wayland client" 1.22.0 "" "-lwayland-client"
emit_pc wayland-cursor "Wayland cursor" 1.22.0 "wayland-client" "-lwayland-cursor"
emit_pc wayland-egl "Wayland EGL" 1.22.0 "wayland-client" "-lwayland-egl"
emit_pc wayland-egl-backend "Wayland EGL backend" 1.22.0 "wayland-egl" ""
emit_pc wayland-server "Wayland server" 1.22.0 "" "-lwayland-server"
emit_pc wayland-scanner "Wayland scanner" 1.22.0 "" ""

# ── 预检：任一项不过立即退出，避免跑满 8 分钟才报错 ──
for pc in x11 wayland-client wayland-egl gobject-2.0 fribidi glib-2.0 gio-2.0 gtk4; do
  pkg-config --exists "$pc" || { echo "pkg-config 缺少: $pc"; exit 1; }
done
CF=$(pkg-config --cflags x11 wayland-client gtk4)
echo "$CF" | grep -q "opt/inc" || { echo "独立 include 未生效"; exit 1; }
if echo "$CF" | grep -q -- "-I/usr/include "; then echo "系统 include 泄漏"; exit 1; fi
[ -f /opt/inc/X11/Xlib.h ] || { echo "Xlib.h 未就位"; exit 1; }
[ -f /opt/inc/wayland-client.h ] || { echo "wayland-client.h 未就位"; exit 1; }

# ── alpine 把库放在 /usr/lib，zig 的 glibc 目标只搜 multiarch 目录 ──
mkdir -p /usr/lib/x86_64-linux-gnu
cd /usr/lib
for f in *.so *.so.*; do
  [ -e "$f" ] || continue
  [ -e "/usr/lib/x86_64-linux-gnu/$f" ] || ln -s "/usr/lib/$f" "/usr/lib/x86_64-linux-gnu/$f"
done
ln -sf /usr/lib/libgtk-4.so.1 /usr/lib/libgtk4.so

# ── 编译：按 glibc 2.35 目标链接 ──
cd /source
/opt/zig/zig build --release=fast -Dtarget=x86_64-linux-gnu.2.35
```

### 运行编译

```bash
# 用带 PID 的新目录，避免之前容器留下的 root 属主文件（rm -rf 删不掉）
WORK=/tmp/gtbuild-$$
mkdir -p "$WORK"
cp -r /tmp/ghostty-src/. "$WORK"/
echo "$WORK" > /tmp/gtwork-path

docker rm -f ghostty-builder 2>/dev/null
docker run --name ghostty-builder --network host \
  -v "$HOME/opt/zig013/zig-linux-x86_64-0.13.0:/opt/zig:ro" \
  -v "$HOME/.cache/zig-persist:/root/.cache/zig" \
  -v "$WORK:/source" \
  -v /tmp/build-ghostty.sh:/build.sh:ro \
  alpine:3.20 sh /build.sh > /tmp/gtbuild.log 2>&1
```

**关键参数说明**：

- `alpine:3.20`（不能用 `latest`）：3.24 已移除 `wayland-dev` 包
- `--network host`：容器共用宿主网络栈
- 挂载缓存卷：zig 依赖缓存跨容器保留
- **不要用 `--rm`**：需要事后 `docker cp` 取产物
- **不要挂 `:ro`**：zig 要写 `.zig-cache`

### 取出产物

```bash
mkdir -p ~/github/ghostty-ubuntu22.04/out/bin
cp "$WORK/zig-out/bin/ghostty" ~/github/ghostty-ubuntu22.04/out/bin/ghostty
chmod +x ~/github/ghostty-ubuntu22.04/out/bin/ghostty

# 验证：glibc 引用应止于 2.35
readelf -lW ~/github/ghostty-ubuntu22.04/out/bin/ghostty | grep -A1 INTERP | tail -1
strings -a ~/github/ghostty-ubuntu22.04/out/bin/ghostty | grep -E "^GLIBC_2\." | sort -V -u | tail -3
```

## 阶段三：捆绑 GTK 依赖库

22.04 自带的 GTK 4.0 是 4.6，ghostty 1.1.3 需要 4.14+。从 Ubuntu 24.04 (noble) 的 deb 提取并 patchelf 重定向。

```bash
cd ~/github/ghostty-ubuntu22.04
env PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
  make all_libs patch_bin
```

### `fixver2.py` 的作用

noble 的库引用 `GLIBC_2.38`，22.04 只有到 2.35。需要同时改两处，缺一不可：

1. `.dynstr` 里的版本名字符串 `GLIBC_2.38` → `GLIBC_2.35`（等长，偏移不变）
2. `.gnu.version_r` 中对应 vernaux 的 `vna_hash`（glibc 先比对哈希再比对字符串）

`libproxies.so` 提供缺失的符号本体：`__isoc23_*` 转发到旧函数，`strlcpy`/`strlcat` 用 OpenBSD 实现，`fmod`/`fmodf` 用 `dlvsym` 转发到 `GLIBC_2.2.5`（noble 对 libm 兼容符号做了重新版本化）。

## 阶段四：修复相对路径（必做）

**这一步漏了会导致「点桌面图标没反应」**。

`make` 里所有库都是用 `--add-needed $(LIB_DIR)/libproxies.so` 加的，展开成**相对路径** `./lib/libproxies.so`。这只在当前工作目录恰好是项目根目录时成立——从终端测试容易误判为成功，但 GNOME 从图标启动时 cwd 是 home，直接失败。

```bash
cd ~/github/ghostty-ubuntu22.04
LIBABS=$(realpath lib)

# 1. 把库内部的相对引用换成绝对路径
for f in lib/*.so; do
  ./patchelf --remove-rpath "$f" 2>/dev/null
  ./patchelf --set-rpath "$LIBABS" "$f" 2>/dev/null
  ./patchelf --replace-needed ./lib/libproxies.so "$LIBABS/libproxies.so" "$f" 2>/dev/null
done

# 2. 给二进制设置 RUNPATH
./patchelf --set-rpath "$LIBABS" out/bin/ghostty

# 验证
for f in lib/*.so; do
  readelf -dW "$f" | grep -c "\./lib/" | grep -qv '^0$' && echo "❌ $(basename $f) 仍有相对引用"
done
readelf -dW out/bin/ghostty | grep -E "RPATH|RUNPATH"
```

## 阶段五：安装

```bash
cd ~/github/ghostty-ubuntu22.04
BIN=$(realpath out/bin/ghostty)

# 命令链接
mkdir -p ~/.local/bin
ln -sf "$BIN" ~/.local/bin/ghostty

# 资源文件
docker cp ghostty-builder:/source/zig-out/share out/ 2>/dev/null

# terminfo
mkdir -p ~/.terminfo/x
cp out/share/terminfo/x/xterm-ghostty ~/.terminfo/x/
TERMINFO=~/.terminfo infocmp xterm-ghostty >/dev/null && echo "terminfo OK"

# 桌面文件
mkdir -p ~/.local/share/applications
sed "s|^Exec=.*|Exec=$BIN|" \
  out/share/applications/com.mitchellh.ghostty.desktop \
  > ~/.local/share/applications/com.mitchellh.ghostty.desktop

# 图标
mkdir -p ~/.local/share/icons
cp -r out/share/icons/hicolor ~/.local/share/icons/
gtk-update-icon-cache ~/.local/share/icons/hicolor 2>/dev/null
update-desktop-database ~/.local/share/applications 2>/dev/null
```

## 阶段六：验证

**必须用最小环境测试**，否则会像本次一样误判成功：

```bash
# 从任意目录（模拟 GNOME 从图标启动）
cd ~
env -i DISPLAY=:1 HOME=$HOME PATH=/usr/bin:/bin \
  ~/github/ghostty-ubuntu22.04/out/bin/ghostty > /tmp/g.log 2>&1 &
sleep 6

# 确认窗口真的创建了
xwininfo -root -tree | grep -i ghostty
pkill -f "out/bin/ghostty"
```

判定标准：

- `xwininfo` 能列出 `("ghostty" "com.mitchellh.ghostty")` 窗口 → 成功
- 只有进程存活但无窗口 → 不算成功
- 日志有 `error while loading shared libraries` → 库路径问题

## 已知陷阱汇总

### 1. zig TLS 不走系统 CA

`zig build` 报 `TlsInitializationFailed`，但同时 `curl` 完全正常。zig 用自带的 CA 证书集，不读 `/etc/ssl/certs/ca-certificates.crt`。

**解法**：用 curl 下载所有依赖，再 `zig fetch` 喂进缓存。

### 2. pkgconf 过滤系统 include 路径

alpine 的 `x11.pc` 写 `Cflags: -I${includedir}`，pkgconf 展开后为空。改成 `-I/usr/include` 也不行——pkgconf 会当系统路径吃掉；即使强制保留，也会让 zig 优先读到 musl 头（`sys/cdefs.h` 报 `non-standard #include is deprecated`），与 zig 自带 glibc 头冲突。

**解法**：把 X11/wayland 头复制到 `/opt/inc/`，重写 `.pc` 只指向该独立目录。

### 3. alpine 库路径与 glibc multiarch 不一致

库在 `/usr/lib/`，但 zig 的 glibc 目标只搜 `/usr/lib/x86_64-linux-gnu/`，报 `unable to find dynamic system library`。

**解法**：批量建软链到 multiarch 目录。

### 4. `alpine:latest` 会漂移

3.24 已移除 `wayland-dev` 包；仓库里的 zig 是 0.16（schema 不兼容）。必须钉住 `alpine:3.20`。

### 5. `patchelf --set-interpreter` 会让 glibc 二进制崩溃

原 Makefile 里有这步（给 musl 用的）。glibc 编译的产物解释器本来就是 `/lib64/ld-linux-x86-64.so.2`，重设后动态链接器在 `elf_setup_debug_entry` 段错误。

**解法**：从 `patch_bin` 目标里移除。

### 6. 容器留下 root 属主文件

容器内以 root 写入，宿主机上 `rm -rf` 会报权限不够，而 sudo 需要密码。

**解法**：用带 PID 的新目录，或从容器内清理。

### 7. apk 索引缓存损坏

`apk update` 报 `Permission denied` 但 CDN 实际可达。

**解法**：`rm -rf /var/cache/apk/*` 后重试。

### 8. `make ... | tee` 的退出码不可信

管道的退出码是 `tee` 的，make 早已失败却会报 exit 0。

**解法**：用 `> /tmp/log 2>&1` 完整重定向，直接看日志内容判断。

## 环境固化建议

`/tmp` 会被清理，重建前需重新准备：

- `~/opt/zig013/` — zig 0.13.0（约 400MB）
- `~/.cache/zig-persist/` — 依赖缓存（约 517MB）
- `/tmp/build-ghostty.sh` — 构建脚本
- `/tmp/ghostty-src/` — 源码

建议把这四项移到 `~/github/ghostty-ubuntu22.04/` 下的持久位置。
