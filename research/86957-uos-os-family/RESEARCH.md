# Research: UOS Server should be classified as RedHat family

Issue: <https://github.com/ansible/ansible/issues/86957>

> **Status:** WIP — research notes & code in progress. Not yet a formal PR.
> This branch lives on a personal fork for backup / sharing only.

## TL;DR

`ansible_facts['os_family']` currently classifies **all** UnionTech / UOS
distributions as `Debian`, because of the logic added by PR
[#77275](https://github.com/ansible/ansible/pull/77275). That is wrong for
**UOS Server**, which is RPM-based and built on top of openAnolis (RHEL
derivative) or openEuler.

The proposal: keep UOS desktop as `Debian`, but classify UOS Server as
`RedHat`. Detection key: `PLATFORM_ID="platform:uel*"` in `/etc/os-release`
(value is set on every Server image we surveyed; never set on Desktop).

A previous attempt — PR [#79290](https://github.com/ansible/ansible/pull/79290)
— was closed because it broke backward compatibility for desktop users.
This proposal avoids that by gating purely on `PLATFORM_ID`, so existing
UOS Desktop fixtures (no `PLATFORM_ID`) keep their `Debian` classification.

## UOS Server flavours (A / E / D / C)

UnionTech publishes UOS Server in four flavours, each based on a different
upstream:

| Flavour | Upstream | Family | Package manager |
|---------|----------|--------|-----------------|
| **A** version | openAnolis (Alibaba, CentOS-derived) | RHEL | rpm / yum / dnf |
| **E** version | openEuler (Huawei) | RHEL | rpm / yum / dnf |
| **D** version | Debian | Debian | dpkg / apt |
| **C** version | CentOS | RHEL | rpm / yum / dnf |

Sources:

- UnionTech official knowledge base "ADE 版本区别介绍":
  <https://faq.uniontech.com/solution/umountain>
- 博客园 — "统信(UOS)服务器版本 A E D C 4个版本":
  <https://www.cnblogs.com/zmbhfly/p/17853005.html>
- 知乎 — "统信UOS V20 1050三个版本(1050a, 1050d, 1050e)有什么区别":
  <https://www.zhihu.com/question/542094510>

### `VERSION_CODENAME` mapping

UOS Server uses Chinese cultural codenames (pinyin) in `VERSION_CODENAME`:

| `VERSION_CODENAME` | 中文 | Meaning | Maps to |
|--------------------|------|---------|---------|
| `kongzi` | 孔子 | Confucius | **A** version (openAnolis) |
| `fuyu`   | 伏羲 | Fuxi (legendary sage-king) | **E** version (openEuler) |

The naming acronyms in `PLATFORM_ID` align with this:

- `uel` = **U**OS **E**nterprise **L**inux (≈ RHEL)
- `uelc` = `uel` + **c**loud (Anolis is cloud-native, hence the `c`)

## Survey of real images

We pulled 12 real UOS Server images from Docker Hub (multiple builders,
multiple point releases) and dumped `/etc/os-release` plus related files
to confirm what fields are reliably set in the wild. Raw dumps are in
[`raw-os-release/`](raw-os-release/).

| Image | Suffix | `PLATFORM_ID` | `VERSION_CODENAME` | `NAME` |
|-------|--------|---------------|--------------------|--------|
| `cnxc/uos:server-20-1050a` | a | `platform:uelc20` | `kongzi` | UnionTech OS Server 20 |
| `cnxc/uos:server-20-1060a` | a | `platform:uelc20` | `kongzi` | UOS Server 20 |
| `cnxc/uos:server-20-1070a` | a | `platform:uelc20` | `kongzi` | UOS Server 20 |
| `liwanggui/uos-server:v20-1060a` | a | `platform:uelc20` | `kongzi` | UOS Server 20 |
| `xudingjun3131/uos:1050u1a` | a (update1) | `platform:uelc20` | `kongzi` | UnionTech OS Server 20 |
| `xudingjun3131/uos:1050e` | e | `platform:uel20` | `fuyu` | UnionTech OS Server 20 |
| `xudingjun3131/uos:1050u1e` | e (update1) | `platform:uel20` | `fuyu` | UnionTech OS Server 20 |
| `xudingjun3131/uos:1060e` | e | `platform:uel20` | `fuyu` | UOS Server 20 |
| `quanshengli/uniontechos-server-20-1050e:v20` | e | `platform:uel20` | `fuyu` | UnionTech OS Server 20 |
| `macrosan/uos:v20-1050` | (none) | `platform:uel20` | `fuyu` | UnionTech OS Server 20 |
| `macrosan/uos:v20-1060` | (none) | `platform:uel20` | `fuyu` | UOS Server 20 |
| `macrosan/uos:v20-1070` | (none) | `platform:uel20` | `fuyu` | UOS Server 20 |

### Confirmed invariants

1. **All 12 images are RPM-based** — every one of them has `rpm` / `yum` / `dnf`.
2. **`PLATFORM_ID` perfectly distinguishes Server from Desktop**:
   - Server (RHEL family) always has `PLATFORM_ID="platform:uel*"` (`uel20` for E, `uelc20` for A).
   - Desktop (Debian family, existing fixture `uos_20.json`) has no `PLATFORM_ID`.
3. **Codename ↔ suffix is 1:1**:
   - `kongzi` ⇔ A version (openAnolis) ⇔ `uelc20`
   - `fuyu`   ⇔ E version (openEuler) ⇔ `uel20`
4. **`/etc/redhat-release` is NOT reliable** as a detection trigger:
   - A-version images symlink `/etc/redhat-release -> uos-release` (works).
   - E-version images do **not** ship `/etc/redhat-release` at all — only `/etc/UnionTech-release`.
   - **Detection must rely on `/etc/os-release` + `PLATFORM_ID`, not the redhat-release file.**

We did not find any `d`-suffix Server image on Docker Hub (Debian-based
Server is the rarer flavour). The existing `uos_20.json` fixture (no
`PLATFORM_ID`) acts as the Debian fallback and stays classified as
`Debian` — preserving backward compatibility, which was the reason
PR #79290 got closed.

## Why not identify A-version as `Anolis OS` and E-version as `openEuler`?

`kongzi` (孔子) is conceptually closer to openAnolis, and `fuyu` (伏羲) is
closer to openEuler — so it's tempting to set
`ansible_distribution = "Anolis OS"` for the A version and
`ansible_distribution = "openEuler"` for the E version. We deliberately
do **not** do that. Reasons:

1. **Brand identity matches existing precedent.** Rocky Linux and AlmaLinux
   are both RHEL rebuilds, yet ansible identifies them as `Rocky` and
   `AlmaLinux` (not `RedHat`). openEuler itself is downstream of
   CentOS Stream, yet ansible identifies it as `openEuler` (not `CentOS`).
   UOS Server is the same situation — UnionTech's own product, built on top
   of openAnolis/openEuler. It deserves its own brand name in
   `ansible_distribution`, just like Rocky/AlmaLinux/openEuler do.
2. **`/etc/os-release` says so.** All 12 surveyed images have `ID=uos`
   and `NAME="UnionTech OS Server 20"` (or `"UOS Server 20"`). They
   identify themselves as UnionTech, not Anolis or openEuler.
3. **`Anolis OS` is not in `OS_FAMILY_MAP`.** Setting
   `distribution = "Anolis OS"` would actually leave
   `os_family = "Anolis OS"` (the fallback), which is worse than the
   current bug. Fixing Anolis is out of scope for this PR.
4. **Information is preserved via `ansible_distribution_release`.** Users
   who need to distinguish A vs. E variants can do so cleanly:

   ```yaml
   when: ansible_distribution_release == 'fuyu'   # E version (openEuler-based)
   when: ansible_distribution_release == 'kongzi' # A version (Anolis-based)
   ```

   This is exactly what `VERSION_CODENAME` is designed for — and the codename
   is set by UnionTech themselves, so it's authoritative.

So the final mapping this PR implements is:

```
ansible_distribution         = "UnionTech"
ansible_distribution_version = "20"
ansible_distribution_release = "kongzi"  # or "fuyu", from /etc/os-release
ansible_os_family            = "RedHat"
```

## History — why earlier attempts failed

- **PR #77275** (merged) — added the current `Uos` → `Debian` mapping.
  Correct for desktop, wrong for server.
- **PR #79290** (closed by maintainer @sivel) — tried to put all UnionTech
  under RedHat. Closed because it broke backward compatibility for
  desktop users.

This proposal threads the needle by gating on `PLATFORM_ID`, which is
**only ever set on Server images**, leaving Desktop behaviour untouched.

## Proposed change (in this branch)

`lib/ansible/module_utils/facts/system/distribution.py`:

1. Add two `OSDIST_LIST` entries for UnionTech:
   - `/etc/redhat-release` (A-version path)
   - `/etc/os-release` (E-version path, ordered before the existing
     `Debian` entry so it wins when `PLATFORM_ID="platform:uel*"`)
2. Add `parse_distribution_file_UnionTech` that returns success only
   when `PLATFORM_ID` starts with `platform:uel`, OR the
   `/etc/redhat-release` content contains `UnionTech OS Server` /
   `UOS Server`.
3. In the existing `parse_distribution_file_Debian` UOS branch, return
   `False` early if `PLATFORM_ID="platform:uel*"`, so the new
   UnionTech parser takes over.
4. Append `'UnionTech'` to `OS_FAMILY_MAP['RedHat']`.

### Test fixtures added

- `uniontech_os_server_20_uelc20.json` — A version (kongzi / uelc20),
  modeled on `cnxc/uos:server-20-1050a`.
- `uniontech_os_server_20_uel20.json` — E version (fuyu / uel20),
  modeled on `quanshengli/uniontechos-server-20-1050e:v20`.
- `uos_server_20_uelc20.json` — `NAME="UOS Server 20"` variant
  (vs. `"UnionTech OS Server 20"`), modeled on `cnxc/uos:server-20-1070a`.

The existing `uos_20.json` fixture (Desktop, no `PLATFORM_ID`) is left
untouched and continues to be classified as `Debian`. Unit tests pass:
**97 passed**.

## TODO before opening upstream PR

- [ ] Get `ansible-test sanity` running locally and clean.
- [ ] Add `changelogs/fragments/86957-uniontechos-redhat-family.yml`.
- [ ] Squash, sign (`-s -S`), push to fork, open PR with `Fixes #86957`
      in the description (not the commit message — Prow-style etiquette).
