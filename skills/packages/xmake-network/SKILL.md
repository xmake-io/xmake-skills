---
name: xmake-network
description: Use when package downloads are slow, failing, or blocked — especially in China or behind corporate networks. Covers HTTP/SOCKS5 proxies (`xmake g --proxy`), host allow-lists, `pac.lua` routing, `mirror()` URL rewriting for GitHub acceleration, manual download directories, and recommended mirror setups for domestic network environments.
---

# Xmake Network & Download Acceleration

When `xrepo install`, `xmake require`, or package fetches are slow or fail, xmake gives you several tools in increasing order of power: manual download dir → global proxy → proxy with host allow-list → `pac.lua` for full rule-based routing and URL mirroring.

All of these are configured with `xmake g` (global config) — settings persist across all projects on the machine.

## 1. Quickest win: manual download directory

If you already have the source tarballs (downloaded by hand, from a mirror, or shared across a team), drop them in a directory and point xmake at it:

```bash
xmake g --pkg_searchdirs="/download/packages"
```

Xmake will search this directory **before** hitting the network. To find out what filename to put there:

```bash
xmake require --info zlib
# -> searchnames: zlib-1.2.11.tar.gz
```

Drop `zlib-1.2.11.tar.gz` into `/download/packages/` and the next install uses it offline.

Good for: CI caches, air-gapped machines, shared team caches on a file server.

## 2. Global proxy

```bash
xmake g --proxy="http://127.0.0.1:7890"
xmake g --proxy="https://host:port"
xmake g --proxy="socks5://127.0.0.1:1086"
```

This proxies **everything** xmake downloads — git, curl, wget, HTTP packages. Unset with `xmake g --proxy=`.

Notes:

- The protocol is passed through to the underlying downloader. `socks5://` only works for curl/git — `wget` does not support socks5, so packages that xmake fetches via wget may fail. Prefer `http://` or `https://` proxies for maximum compatibility.
- `xmake g --help` lists all network-related flags.

## 3. Proxy only for specific hosts

Global proxying slows down non-blocked traffic. Scope it down:

```bash
xmake g --proxy="http://127.0.0.1:7890" \
        --proxy_hosts="github.com,*.githubusercontent.com,gitlab.*,*.xmake.io"
```

Only matching hosts go through the proxy; everything else goes direct. `*` is a Lua pattern wildcard (more flexible than shell glob — any Lua pattern works).

Unset with `xmake g --proxy_hosts=`.

## 4. `pac.lua` — rule-based proxy + URL mirroring

For anything more complex than "proxy these hosts", use a `pac.lua` (proxy auto-config) file. Default location:

```
~/.xmake/pac.lua
```

Xmake automatically picks it up if `--proxy` is set and the file exists. Override the location with `xmake g --proxy_pac=/abs/path/pac.lua`.

### `main(url, host)` — decide whether to proxy

Return `true` to route through the proxy, `false` or nothing to go direct:

```lua
-- ~/.xmake/pac.lua
function main(url, host)
    if host == "github.com" then
        return true
    end
    if host:find("githubusercontent.com", 1, true) then
        return true
    end
    if host:find("gitlab%.") then
        return true
    end
    -- everything else: direct
    return false
end
```

- Precedence: if both `--proxy_hosts` and `pac.lua` exist, **`--proxy_hosts` wins**. Remove `--proxy_hosts` if you want `pac.lua` to take over.
- The file is plain Lua — `if`, string patterns, and tables all work.

### `mirror(url)` — rewrite URLs for acceleration

Available since v2.5.4. The function rewrites download URLs before xmake hits the network — use it to swap `github.com` for a mirror that is fast in your region:

```lua
-- ~/.xmake/pac.lua
function mirror(url)
    -- GitHub → a fast mirror (pick one that works for you)
    url = url:gsub("github%.com", "kkgithub.com")
    url = url:gsub("raw%.githubusercontent%.com", "raw.kkgithub.com")
    return url
end
```

After:

```bash
xrepo install libpng
# -> curl https://kkgithub.com/glennrp/libpng/archive/v1.6.37.zip ...
```

`mirror` and `main` coexist: `mirror` rewrites the URL, then `main` decides whether to proxy the rewritten host.

### Combined example (recommended for China network)

```lua
-- ~/.xmake/pac.lua

-- Rewrite slow/blocked URLs to fast mirrors
function mirror(url)
    -- GitHub archives and raw content
    url = url:gsub("https://github%.com", "https://kkgithub.com")
    url = url:gsub("https://raw%.githubusercontent%.com", "https://raw.kkgithub.com")
    url = url:gsub("https://codeload%.github%.com", "https://codeload.kkgithub.com")
    return url
end

-- Fall back to proxy for hosts that have no good mirror
function main(url, host)
    local proxied = {
        ["sourceforge.net"] = true,
        ["www.sourceforge.net"] = true,
        ["download.qt.io"] = true,
    }
    if proxied[host] then
        return true
    end
    return false
end
```

With this file and `xmake g --proxy="http://127.0.0.1:7890"` set, GitHub traffic uses the fast mirror directly, SourceForge/Qt fall through to your local proxy, and everything else goes direct.

## 5. Repository mirror for `xmake-repo` itself

The **recipe repository** (`xmake-repo`, where `add_requires` resolves names) is cloned to `~/.xmake/repositories/xmake-repo`. If cloning from GitHub is slow:

```bash
# Switch to the Gitee mirror of xmake-repo
xmake g --repositories="xmake-repo https://gitee.com/tboox/xmake-repo.git"

# Or Gitea/self-hosted mirror
xmake g --repositories="xmake-repo https://my-gitea.example.com/mirror/xmake-repo.git"
```

Verify / reset:

```bash
xmake repo --list
xmake repo --clean        # wipe cloned repos, forces re-clone next run
xmake repo --update       # update all configured repos now
```

## 6. Binary-package cache (fastest path for common deps)

Xmake supports binary package caches (precompiled archives). Many `xmake-repo` packages ship binaries — set this up to skip local compilation entirely for common deps:

```bash
xmake g --pkg_installdir=~/.xmake/packages        # default; installed packages
```

To force a package to try the precompiled binary:

```bash
xrepo install --precompiled fmt
```

In CI, cache `~/.xmake/packages` and `~/.xmake/cache` between runs for instant second builds.

## 7. Inspecting the network layer

```bash
xmake g --show                         # show all global config including proxy
xmake require --info <pkg>             # resolved URLs, searchdirs, searchnames
xmake require --force <pkg>            # rebuild/re-download
xmake -vD                              # verbose + diagnosis — shows actual download commands
XMAKE_PROFILE=trace xmake ...          # trace every subprocess (incl. curl/git)
```

If a download hangs, use `XMAKE_PROFILE=stuck` to see which URL xmake is blocked on — see `xmake-troubleshooting`.

## Decision tree

```
Package downloads failing / slow?
│
├── Tarballs already on disk  → xmake g --pkg_searchdirs=...
│
├── One global proxy is fine  → xmake g --proxy=http://host:port
│
├── Only specific hosts blocked
│     └─ xmake g --proxy=... --proxy_hosts="github.com,*.githubusercontent.com,..."
│
├── Need per-URL rewriting / mirror acceleration
│     └─ write ~/.xmake/pac.lua with main()+mirror()
│
├── xmake-repo clone itself is slow
│     └─ xmake g --repositories="xmake-repo https://gitee.com/tboox/xmake-repo.git"
│
└── Nothing works / hang
      └─ XMAKE_PROFILE=stuck xmake ... → see xmake-troubleshooting
```

## Pitfalls

- **`socks5://` + wget.** Packages downloaded via wget break. Prefer `http://` proxies, or install a SOCKS-to-HTTP bridge (e.g. `privoxy`).
- **`--proxy_hosts` overrides `pac.lua`.** If your `main()` isn't being called, check whether `--proxy_hosts` is set — it takes precedence.
- **Lua patterns, not glob.** `*.github.com` in `--proxy_hosts` is a Lua pattern; `*` matches zero-or-more of the previous character (usually fine for hosts, but not shell semantics). Test with `xmake -vD` if unsure.
- **Stale cache hiding a fix.** After changing proxy/mirror, `xmake require --force <pkg>` to re-download.
- **Mirror rewrites break hash verification if the mirror serves different bytes.** Only use mirrors that are exact byte-for-byte copies of upstream (kkgithub, ghfast.top, etc. are — random re-packagers are not).
- **Forgetting to set the proxy globally for xmake-repo clones.** The recipe repo is cloned with `git`, which also needs the proxy. `git config --global http.proxy ...` or set it in `pac.lua` via `--proxy`.

## When to branch out

- Installing / fetching packages → `xrepo-cli`, `xmake-packages`
- Hang diagnosis during download → `xmake-troubleshooting`
- Private/internal package repos → `xmake-private-packages`
