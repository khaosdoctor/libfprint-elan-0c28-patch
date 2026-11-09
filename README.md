# libfprint-elan-0c28-patch

Patched libfprint for Arch Linux that makes the **ELAN `04f3:0c28`** USB fingerprint reader (sold as the FeinTech FPS00200) actually work with fprintd.

> [!IMPORTANT]
> This package is built for that one sensor and nothing else. The patch only changes behavior when libfprint finds a device with USB ID `04f3:0c28`, every other reader keeps the stock driver code. If your reader has a different ID, this package won't help you and you should stay on Arch's `libfprint`.

You can check your ID with:

```bash
lsusb | grep 04f3:0c28
```

## The problem

Stock libfprint lists `04f3:0c28` as supported, and fprintd finds it, but you'll never get a match. Enrolling works (sometimes it fails with `Calibration failed!`), then every `fprintd-verify` comes back `verify-no-match` with a score of 0.

The `elan` driver treats every ELAN sensor as a swipe sensor and stitches frames together based on how the finger moves, but this one is a 96x96 square pad you press and hold. The stitched images come out garbled, and even after fixing the capture, a 96x96 image (about 5x5 mm of fingertip) has too few minutiae for libfprint's NBIS matcher to ever score a match. [docs/how-it-works.md](docs/how-it-works.md) has the details.

This package fixes both: it captures a single sharp frame per touch, and it swaps NBIS for SIGFM, a texture-based matcher written for tiny sensors.

## Installing

You need `base-devel`. The PKGBUILD is Arch's own `libfprint` PKGBUILD with the patch added, so the build and the test suite are the same as the official package.

```bash
git clone https://github.com/khaosdoctor/libfprint-elan-0c28-patch.git
cd libfprint-elan-0c28-patch
gpg --keyserver hkps://keyserver.ubuntu.com --recv-keys D4C501DA48EB797A081750939449C2F50996635F
makepkg -si
```

pacman will ask to remove `libfprint`, say yes. This package provides `libfprint`, so fprintd keeps working, and since it has a different name `pacman -Syu` and `yay -Syu` won't put the stock one back over it.

After installing, restart fprintd and enroll again. Any print you enrolled with the stock driver won't match anymore (more on that in [troubleshooting](#troubleshooting)).

```bash
sudo systemctl restart fprintd
fprintd-delete $USER
fprintd-enroll
fprintd-verify
```

Enrolling takes **15 touches**. Hold your finger still for about a second each time, lift it all the way, and move it a bit between touches (centre, tip, left, right). The sensor only sees a small patch of your finger, so the more of it you cover during enroll, the better verify works later.

## Updating

When Arch ships a new libfprint, this package won't update by itself. Bump `pkgver` in the PKGBUILD, update the first `b2sums` entry (use `makepkg -g`), check that the patch still applies, and rebuild:

```bash
makepkg -si
```

If the patch doesn't apply anymore, upstream changed `elan.c` or the image device code and the patch needs a refresh.

## How well it works

Tested with one person and one sensor, so take these as a rough idea rather than a benchmark:

| | Attempts | Result |
|---|---|---|
| Enrolled finger | 6 | 4 matched (~2 out of 3 per touch) |
| Other fingers | 9 | all rejected, highest score 3 out of 7 needed |

A rejected touch just means you touch again. `pam_fprintd` gives you 3 tries by default, so for sudo and login you'll fall back to your password rarely.

> [!WARNING]
> This is an image sensor: your fingerprint templates are stored on disk in `/var/lib/fprint/`, readable by root. The impostor testing above is super small. Keep a password on your account, and I wouldn't use this as the only thing protecting anything that matters.

## Troubleshooting

**`enroll-unknown-error` and `Calibration failed!` in the logs.** The sensor calibrates with no finger on it before every touch. If your finger is still on the pad when the next touch starts, calibration fails. Lift your finger all the way between touches and run `fprintd-enroll` again.

**`verify-no-match` on every try, with your right finger.** If you enrolled before installing this package, your stored print is an NBIS print and this driver can't use it. The fprintd log says `stored print has no SIGFM data, re-enroll this finger`. Run `fprintd-delete $USER` and enroll again.

**`verify-no-match` some of the time.** That's expected with a sensor this small, see the numbers above. If it fails most of the time, re-enroll and spread your 15 touches over more of your finger, it's usually because the touch lands on a patch you never enrolled.

**`verify-swipe-too-short` or `enroll-swipe-too-short`.** You lifted your finger before the sensor got enough frames. Hold it down for about a second. fprintd still calls it a swipe because that's the only retry message it has.

**`enroll-retry-scan` once in a while.** The driver throws away captures that look like sensor noise (vertical stripes instead of ridges) or that have too few features to match. Touch again.

**`makepkg` fails with `unknown public key 40F65066AD7C16DB`.** That's a signing subkey Marco Trevisan (the libfprint maintainer) added in 2025, and older copies of his key don't have it. Fetch his key again with the `gpg --recv-keys` line from [installing](#installing).

**`makepkg` fails at `FPrint-2.0.gir` with `No module named 'distutils'`.** `g-ir-scanner` runs whatever `python3` comes first in your `PATH`, and if that's a pyenv, mise or asdf Python it won't have setuptools. Build with the system Python first:

```bash
PATH=/usr/bin:$PATH makepkg -si
```

**Reading the logs.** If something else breaks, run fprintd by hand with debug output and look at the `elan` and `sigfm` lines:

```bash
sudo systemctl stop fprintd
sudo G_MESSAGES_DEBUG=all /usr/lib/fprintd -t 2>&1 | grep -E 'elan|sigfm|error'
```

Then run `fprintd-verify` in another terminal. Each touch logs the frame it picked, the stripe ratio, the SIGFM keypoint count and the best match score.

## Credits

SIGFM is by Matthieu Charette, Natasha England-Elbro and Timur Mangliev, and I took the pure-C version from [buxel/libfprint](https://github.com/buxel/libfprint) (branch `goodixtls-on-1.94.9`), where it was rewritten for the Goodix GF511. [buxel/libfprint-27c6-5110](https://github.com/buxel/libfprint-27c6-5110) has the benchmarks that the threshold here comes from.

The patch (and SIGFM inside it) is LGPL-2.1-or-later, same as libfprint. The packaging files (PKGBUILD, workflow, docs) are 0BSD like Arch's own packaging, see [LICENSE](LICENSE).

## Publishing

Pushing a tag that matches the PKGBUILD version (`v1.94.100-1` for `pkgver=1.94.100`, `pkgrel=1`) builds and tests the package in an Arch container and pushes it to the AUR. The push needs an `AUR_KEY` repository secret with the private SSH key of an AUR account, and until that secret exists the workflow builds and stops. Remember to regenerate `.SRCINFO` with `makepkg --printsrcinfo > .SRCINFO` when you change the PKGBUILD.
