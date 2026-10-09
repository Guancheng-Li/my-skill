# Compiling and Installing Ghostty on Ubuntu 22.04

## Overview

Build and install the Ghostty terminal emulator from source on Ubuntu 22.04 (jammy, glibc 2.35).

Reference project: `~/github/ghostty-ubuntu22.04` (forked from [Maix0/ghostty-ubuntu22.04](https://github.com/Maix0/ghostty-ubuntu22.04))

### Core difficulty

Ghostty officially ships only for macOS and Linux (glibc 2.38+ / Ubuntu 24.04). Ubuntu 22.04 has glibc 2.35, which lacks the `GLIBC_2.38` symbol versions. This process solves it in three steps: **cross-compile against a glibc 2.35 target + bundle GTK libraries + provide a symbol shim**.

## Requirements

| Item | Requirement | Notes |
|------|-------------|-------|
| OS | Ubuntu 22.04.5 LTS | glibc 2.35 |
| Docker | Usable without sudo | `docker ps` must work |
| Disk | ~5 GB | ~500 MB dependency cache, ~1.8 GB container |
| Network | Reaches github.com / alpine CDN | See [Known Pitfalls](#known-pitfalls) |

## Phase 1: Prepare the Build Environment

### 1. Pin the zig version

Ghostty 1.1.3's `build.zig.zon` uses the zig 0.13-era schema. zig 0.14+ changed how the `.name` field is parsed and fails with `expected enum literal`. **You must use 0.13.0.**

```bash
mkdir -p ~/opt/zig013
cd ~/opt/zig013
curl -L 'https://ziglang.org/download/0.13.0/zig-linux-x86_64-0.13.0.tar.xz' | tar -xJ
# Verify
./zig-linux-x86_64-0.13.0/zig version   # should print 0.13.0
```

### 2. Pre-fetch build dependencies (critical)

**zig's built-in TLS stack does not read the system CA bundle.** A plain `zig build` fails with `TlsInitializationFailed` even though `curl` works fine at the same moment.

Workaround: download every dependency tarball with curl, then feed them into the global cache with `zig fetch`.

```bash
# Download ghostty source
mkdir -p /tmp/ghostty-src && cd /tmp/ghostty-src
curl -L 'https://github.com/ghostty-org/ghostty/archive/refs/tags/v1.1.3.tar.gz' \
    | tar -xz --strip-components=1

# Extract all dependency URLs, download with curl, feed into zig's cache
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
print(f"{len(urls)} dependencies found")
ok = fail = 0
for u in urls:
    dest = "dep.tar.gz"
    for _ in range(4):   # retry on network flakiness
        r = subprocess.run(["/usr/bin/curl","-sL","--max-time","180","-o",dest,u],
                           capture_output=True)
        p = pathlib.Path(dest)
        if r.returncode == 0 and p.exists() and p.stat().st_size > 0: break
        time.sleep(2)
    else:
        print(f"DL-FAIL {u}"); fail += 1; continue
    if subprocess.run([ZIG,"fetch",dest], capture_output=True).returncode == 0: ok += 1
    else: print(f"FETCH-FAIL {u}"); fail += 1
print(f"ok {ok}, failed {fail} / {len(urls)}")
PY

# Keep a copy for the container to mount
rm -rf ~/.cache/zig-persist
cp -a ~/.cache/zig ~/.cache/zig-persist
du -sh ~/.cache/zig-persist   # ~517 MB
```

## Phase 2: Compile in a Container

### Build script

Save as `/tmp/build-ghostty.sh`:

```bash
#!/bin/sh
set -e

# Stale index caches left by earlier containers make apk update report Permission denied
rm -rf /var/cache/apk/* 2>/dev/null || true
apk update -q >/dev/null 2>&1 || true

# The alpine repos intermittently fail TLS; retry until everything lands
for i in 1 2 3 4 5; do
  if apk add --no-cache \
    xz ncurses glib glib-dev libadwaita-dev \
    harfbuzz-static harfbuzz-dev pango-dev gdk-pixbuf-dev \
    gtk4.0 gtk4.0-dev wayland wayland-dev wayland-protocols \
    libx11-dev pkgconf fribidi-dev util-linux-dev >/dev/null 2>&1; then
    break
  fi
  echo "apk add attempt $i failed, retrying..."
  sleep 3
done
# apk add still exits 0 on partial failure, so verify explicitly
apk info -e gtk4.0-dev libx11-dev wayland-dev fribidi-dev >/dev/null 2>&1 \
  || { echo "required packages missing"; exit 1; }

# ── Patch the pkg-config .pc files ──
# alpine's x11.pc / wayland-*.pc write `Cflags: -I${includedir}`, which pkgconf
# expands to nothing. Expanding it to -I/usr/include doesn't work either: pkgconf
# strips system paths, and even when forced through, zig would then prefer the
# musl headers (sys/cdefs.h) over its own bundled glibc headers.
# Compromise: copy the X11/wayland headers into a standalone directory.
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

# ── Preflight: bail out early instead of failing 8 minutes in ──
for pc in x11 wayland-client wayland-egl gobject-2.0 fribidi glib-2.0 gio-2.0 gtk4; do
  pkg-config --exists "$pc" || { echo "pkg-config missing: $pc"; exit 1; }
done
CF=$(pkg-config --cflags x11 wayland-client gtk4)
echo "$CF" | grep -q "opt/inc" || { echo "standalone include not applied"; exit 1; }
if echo "$CF" | grep -q -- "-I/usr/include "; then echo "system include leaked"; exit 1; fi
[ -f /opt/inc/X11/Xlib.h ] || { echo "Xlib.h not in place"; exit 1; }
[ -f /opt/inc/wayland-client.h ] || { echo "wayland-client.h not in place"; exit 1; }

# ── alpine keeps libraries in /usr/lib; zig's glibc target only searches multiarch dirs ──
mkdir -p /usr/lib/x86_64-linux-gnu
cd /usr/lib
for f in *.so *.so.*; do
  [ -e "$f" ] || continue
  [ -e "/usr/lib/x86_64-linux-gnu/$f" ] || ln -s "/usr/lib/$f" "/usr/lib/x86_64-linux-gnu/$f"
done
ln -sf /usr/lib/libgtk-4.so.1 /usr/lib/libgtk4.so

# ── Compile against a glibc 2.35 target ──
cd /source
/opt/zig/zig build --release=fast -Dtarget=x86_64-linux-gnu.2.35
```

### Run the build

```bash
# Use a fresh PID-suffixed directory to sidestep root-owned files left by
# earlier containers (rm -rf can't delete them without sudo).
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

**Key parameters:**

- `alpine:3.20` (never `latest`): 3.24 dropped the `wayland-dev` package
- `--network host`: container shares the host network stack
- Mounted cache volume: zig dependency cache survives across containers
- **Do not use `--rm`**: you need `docker cp` afterwards to retrieve artifacts
- **Do not mount `:ro`**: zig must write `.zig-cache`

### Retrieve the artifact

```bash
mkdir -p ~/github/ghostty-ubuntu22.04/out/bin
cp "$WORK/zig-out/bin/ghostty" ~/github/ghostty-ubuntu22.04/out/bin/ghostty
chmod +x ~/github/ghostty-ubuntu22.04/out/bin/ghostty

# Verify: glibc references should top out at 2.35
readelf -lW ~/github/ghostty-ubuntu22.04/out/bin/ghostty | grep -A1 INTERP | tail -1
strings -a ~/github/ghostty-ubuntu22.04/out/bin/ghostty | grep -E "^GLIBC_2\." | sort -V -u | tail -3
```

## Phase 3: Bundle the GTK Dependencies

Ubuntu 22.04 ships GTK 4.6; ghostty 1.1.3 needs 4.14+. Extract the libraries from Ubuntu 24.04 (noble) debs and redirect them with patchelf.

```bash
cd ~/github/ghostty-ubuntu22.04
env PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
  make all_libs patch_bin
```

### What `fixver2.py` does

Noble's libraries reference `GLIBC_2.38`; 22.04 only goes up to 2.35. Two places must change together — miss either and loading fails:

1. The version-name string `GLIBC_2.38` → `GLIBC_2.35` in `.dynstr` (same length, so offsets don't shift)
2. The `vna_hash` of the corresponding vernaux entry in `.gnu.version_r` (glibc compares the hash before the string)

`libproxies.so` supplies the missing symbol bodies: `__isoc23_*` forward to the older functions, `strlcpy`/`strlcat` use the OpenBSD implementations, and `fmod`/`fmodf` forward via `dlvsym` to `GLIBC_2.2.5` (noble re-versioned the libm compatibility symbols).

## Phase 4: Fix Relative Paths (required)

**Skipping this is what causes "clicking the desktop icon does nothing".**

Every library in the `make` output gets `--add-needed $(LIB_DIR)/libproxies.so`, which expands to the **relative path** `./lib/libproxies.so`. That only resolves when the current working directory happens to be the project root. Testing from a terminal easily gives a false pass, but GNOME launches apps with cwd set to `$HOME`, so the loader can't find it and the process dies silently.

```bash
cd ~/github/ghostty-ubuntu22.04
LIBABS=$(realpath lib)

# 1. Replace the relative references inside the libraries
for f in lib/*.so; do
  ./patchelf --remove-rpath "$f" 2>/dev/null
  ./patchelf --set-rpath "$LIBABS" "$f" 2>/dev/null
  ./patchelf --replace-needed ./lib/libproxies.so "$LIBABS/libproxies.so" "$f" 2>/dev/null
done

# 2. Give the binary a RUNPATH
./patchelf --set-rpath "$LIBABS" out/bin/ghostty

# Verify
for f in lib/*.so; do
  readelf -dW "$f" | grep -c "\./lib/" | grep -qv '^0$' && echo "FAIL: $(basename $f) still has relative refs"
done
readelf -dW out/bin/ghostty | grep -E "RPATH|RUNPATH"
```

## Phase 5: Install

```bash
cd ~/github/ghostty-ubuntu22.04
BIN=$(realpath out/bin/ghostty)

# Command symlink
mkdir -p ~/.local/bin
ln -sf "$BIN" ~/.local/bin/ghostty

# Resource files
docker cp ghostty-builder:/source/zig-out/share out/ 2>/dev/null

# terminfo
mkdir -p ~/.terminfo/x
cp out/share/terminfo/x/xterm-ghostty ~/.terminfo/x/
TERMINFO=~/.terminfo infocmp xterm-ghostty >/dev/null && echo "terminfo OK"

# Desktop entry
mkdir -p ~/.local/share/applications
sed "s|^Exec=.*|Exec=$BIN|" \
  out/share/applications/com.mitchellh.ghostty.desktop \
  > ~/.local/share/applications/com.mitchellh.ghostty.desktop

# Icons
mkdir -p ~/.local/share/icons
cp -r out/share/icons/hicolor ~/.local/share/icons/
gtk-update-icon-cache ~/.local/share/icons/hicolor 2>/dev/null
update-desktop-database ~/.local/share/applications 2>/dev/null
```

## Phase 6: Verify

**Test with a minimal environment** — otherwise you will repeat the false pass from this build.

```bash
# Launch from an unrelated directory (mimics GNOME starting from the icon)
cd ~
env -i DISPLAY=:1 HOME=$HOME PATH=/usr/bin:/bin \
  ~/github/ghostty-ubuntu22.04/out/bin/ghostty > /tmp/g.log 2>&1 &
sleep 6

# Confirm a window was actually created
xwininfo -root -tree | grep -i ghostty
pkill -f "out/bin/ghostty"
```

Pass criteria:

- `xwininfo` lists a `("ghostty" "com.mitchellh.ghostty")` window → success
- Process alive but no window → not a pass
- Log contains `error while loading shared libraries` → library path problem

## Known Pitfalls

### 1. zig TLS does not use the system CA bundle

`zig build` reports `TlsInitializationFailed` while `curl` works fine. zig carries its own CA set and ignores `/etc/ssl/certs/ca-certificates.crt`.

**Fix**: download dependencies with curl, then `zig fetch` them into the cache.

### 2. pkgconf strips system include paths

alpine's `x11.pc` writes `Cflags: -I${includedir}`, which pkgconf expands to nothing. Rewriting it to `-I/usr/include` doesn't help either — pkgconf treats it as a system path and drops it, and even when forced through, zig then prefers the musl headers (`sys/cdefs.h` → `non-standard #include is deprecated`) over its bundled glibc headers.

**Fix**: copy the X11/wayland headers to `/opt/inc/` and point the rewritten `.pc` files at that standalone directory.

### 3. alpine library layout differs from glibc multiarch

Libraries live in `/usr/lib/`, but zig's glibc target only searches `/usr/lib/x86_64-linux-gnu/`, producing `unable to find dynamic system library`.

**Fix**: batch-create symlinks into the multiarch directory.

### 4. `alpine:latest` drifts

3.24 removed the `wayland-dev` package, and its zig is 0.16 (incompatible schema). Pin `alpine:3.20`.

### 5. `patchelf --set-interpreter` breaks glibc binaries

The original Makefile has this step for the musl build. A glibc-linked binary already uses `/lib64/ld-linux-x86-64.so.2`; rewriting it makes the dynamic linker segfault in `elf_setup_debug_entry`.

**Fix**: remove it from the `patch_bin` target.

### 6. Containers leave root-owned files

The container runs as root, so files it writes can't be removed on the host without sudo — and sudo requires a password here.

**Fix**: use a fresh PID-suffixed directory, or clean up from inside the container.

### 7. Corrupt apk index cache

`apk update` reports `Permission denied` even though the CDN is reachable.

**Fix**: `rm -rf /var/cache/apk/*` and retry.

### 8. `make ... | tee` exit codes are unreliable

The pipeline reports `tee`'s status, so make can fail while the exit code says 0.

**Fix**: redirect with `> /tmp/log 2>&1` and read the log contents.

## Persisting the Environment

`/tmp` gets cleaned periodically. Before rebuilding, restore these four items:

- `~/opt/zig013/` — zig 0.13.0 (~400 MB)
- `~/.cache/zig-persist/` — dependency cache (~517 MB)
- `/tmp/build-ghostty.sh` — build script
- `/tmp/ghostty-src/` — source tree

Consider moving all four into a persistent location under `~/github/ghostty-ubuntu22.04/`.
