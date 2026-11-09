# realspeed-mirrors

Rank Arch Linux and CachyOS pacman mirrors by **downloading real packages** from
each one, instead of trusting a sub-second probe. A drop-in replacement for
`cachyos-rate-mirrors` that also works on plain Arch Linux.

## The problem

pacman always downloads from the **first** mirror in a list and only moves on
when a download *fails*, never when it is slow. So the mirror ranked #1 sets
your update speed.

`cachyos-rate-mirrors` gets that #1 wrong in two ways:

1. **Forced mirrors.** After ranking, it always inserts `mirror.zyner.org` and
   `archlinux.cachyos.org` at the top of the Arch mirrorlist (outside RU/CN),
   whatever their speed for you.
2. **Too-short speed test.** `rate-mirrors` times one small fixed file for
   0.3–1 s. Some mirrors serve that file fast but real packages slowly, and
   no `RATE_MIRRORS_*` setting helps: the test ends when the file does.

A real case (Canada, Nov 2026), where updates crawled at ~600 KB/s per file:

| Mirror | rate-mirrors said | Real package downloads | Rank given |
|---|---|---|---|
| ca.mirror.cx (CachyOS) | 43–56 MB/s | **2.2–3.0 MB/s** on most packages | #1 |
| mirror.rosely.me (CachyOS) | 33 MB/s | 23–40 MB/s | #2 |
| mirror.zyner.org (Arch) | forced, not ranked | **3.1–3.9 MB/s** | #1 |
| mirror.franscorack.com (Arch) | ranked | 10–49 MB/s | #5 |

(Speeds from single test downloads on one connection; they vary between runs.)

## How it works

1. **Candidates:** `rate-mirrors` still finds nearby mirrors (map-aware,
   country-limited), plus a few extra candidates (the CachyOS CDN and the
   mirrors cachyos-rate-mirrors forces). Every candidate gets tested; none is
   forced to the top.
2. **Real test:** it reads a fresh repo database and picks a **random** real
   package (15–100 MB) from the repo you actually use (`extra` for Arch;
   `cachyos-extra-znver4` / `-v4` / `-v3` / `cachyos` for CachyOS). Each mirror
   downloads up to 20 MB of it, one mirror at a time so they don't compete
   for your bandwidth. The 20 MB cap is enforced locally, since some mirrors
   ignore range requests and send the whole file. A random package means no
   mirror gets judged on a file it happens to have cached.
3. **Re-test:** the 5 fastest mirrors download a second random package, and
   each keeps the **lower** of its two speeds. One lucky run can't put a
   mirror on top.
4. **Write:** mirrors are sorted by measured speed, with the speed as a comment.
   Mirrors that fail (missing file, out of sync, unreachable) are dropped. It
   writes `mirrorlist`, `cachyos-mirrorlist` and the existing
   `cachyos-v3/v4-mirrorlist`, and keeps the old files as `*-backup`. If
   nothing could be measured, the lists are left untouched.

It never writes while pacman is running (it waits up to 10 minutes), and only
one instance runs at a time.

## Install

### Release package (recommended)

Install the package from the [latest release](https://github.com/janvitos/realspeed-mirrors/releases/latest).
pacman can install it straight from the URL:

```sh
sudo pacman -U https://github.com/janvitos/realspeed-mirrors/releases/download/v1.0.0/realspeed-mirrors-1.0.0-1-any.pkg.tar.zst
```

### AUR (coming soon)

Not on the AUR yet: new AUR account registration is temporarily closed.
Once it's published, this will work:

```sh
paru -S realspeed-mirrors        # or yay -S realspeed-mirrors
```

### From source

```sh
git clone https://github.com/janvitos/realspeed-mirrors
cd realspeed-mirrors/packaging && makepkg -si
```

### Switch over

```sh
# CachyOS only: stop cachyos-rate-mirrors from overwriting the lists
sudo systemctl mask --now cachyos-rate-mirrors.timer cachyos-rate-mirrors.service

sudo systemctl enable --now realspeed-mirrors.timer   # 1st and 15th of each month
sudo realspeed-mirrors                                 # rank now
sudo pacman -Syyu
```

To go back: `sudo systemctl unmask cachyos-rate-mirrors.timer cachyos-rate-mirrors.service`.

## Usage

```
realspeed-mirrors             rank and write the mirrorlists in /etc/pacman.d (root)
realspeed-mirrors --dry-run   rank and print the result, write nothing (no root needed)
realspeed-mirrors --version
realspeed-mirrors --help
```

Try `realspeed-mirrors --dry-run` first to see how your current #1 mirror really does.

## Configuration

`/etc/conf.d/realspeed-mirrors.conf` (commented defaults included). It also
reads `/etc/conf.d/cachyos-rate-mirrors.env` first, so country settings you
already made still apply.

| Setting | Default | Meaning |
|---|---|---|
| `CANDIDATES` | 10 | mirrors per repo family taken from rate-mirrors |
| `TEST_BYTES` | 20000000 | max bytes per test download |
| `TEST_SECS` | 8 | max seconds per test download |
| `RETEST_TOP` | 5 | fastest mirrors re-tested with a 2nd package (0 = off) |
| `EXTRA_ARCH_MIRRORS` / `EXTRA_CACHYOS_MIRRORS` | CachyOS CDN + forced mirrors | always-tested extra candidates |
| `RATE_MIRRORS_*` | auto country | any rate-mirrors option, e.g. `RATE_MIRRORS_ENTRY_COUNTRY`, `RATE_MIRRORS_EXCLUDE_COUNTRIES` |

The entry country is auto-detected (geoip.kde.org, same as cachyos-rate-mirrors).
In China, neighbours are skipped and more domestic mirrors are tested.

## Costs and limits

- **Data:** about **0.7 GB per run** on CachyOS (measured with systemd
  `IPAccounting`): ~0.2 GB for rate-mirrors' candidate discovery (its probes
  are shortened to 300 ms by default), plus (`CANDIDATES` + `RETEST_TOP`) ×
  20 MB of real tests per repo family. Plain Arch uses about 0.5 GB. At two runs
  a month that's ~1.5 GB/month, close to cachyos-rate-mirrors' own timer.
  On a metered connection, lower `TEST_BYTES` / `CANDIDATES` or run the
  timer less often (`systemctl edit realspeed-mirrors.timer`).
- **Time:** about 1–2 minutes, since mirrors are tested one at a time.
- **Noise:** speeds vary between runs (time of day, mirror load). Taking the
  lower of two tests for the top mirrors damps this, but it's one sample per
  package, not a benchmark.
- **Discovery** still comes from `rate-mirrors`: a fast mirror it never
  outputs won't be tested. Add it to `EXTRA_*_MIRRORS`.
- Anything that calls `cachyos-rate-mirrors` directly (e.g. CachyOS Hello's
  mirror button) still runs the original script and overwrites the lists until
  the next `realspeed-mirrors` run. Run `sudo realspeed-mirrors` instead.

## License

GPL-2.0-or-later, same as cachyos-rate-mirrors.
