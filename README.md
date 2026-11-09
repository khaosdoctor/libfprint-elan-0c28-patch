# libfprint-elan-0c28-patch

Patched libfprint for Arch Linux that makes the **ELAN `04f3:0c28`** USB fingerprint reader (sold as the FeinTech FPS00200) work with fprintd.

> [!IMPORTANT]
> This package is built for that one sensor. Every other reader keeps the stock driver code, so if yours has a different ID, stay on Arch's `libfprint`.

```bash
lsusb | grep 04f3:0c28
```

## The problem

Stock libfprint says it supports this reader, and enrolling works, but verify never matches. The driver treats it as a swipe sensor when it's a small pad you press, and the images it gets are too small for libfprint's matcher anyway. This package captures one sharp frame per touch and matches with SIGFM, which was made for tiny sensors. [docs/how-it-works.md](docs/how-it-works.md) has the details.

## Installing

You need `base-devel`.

```bash
git clone https://github.com/khaosdoctor/libfprint-elan-0c28-patch.git
cd libfprint-elan-0c28-patch
gpg --keyserver hkps://keyserver.ubuntu.com --recv-keys D4C501DA48EB797A081750939449C2F50996635F
makepkg -si
```

pacman will ask to remove `libfprint`, say yes. This package replaces it, so fprintd keeps working and system updates won't put the stock one back.

Then enroll again, prints from the stock driver won't match:

```bash
sudo systemctl restart fprintd
fprintd-delete $USER
fprintd-enroll
```

Enroll asks for a lot of touches. Hold still for a second each time, lift all the way, and move your finger around between touches so you cover more of it.

## Updating

This package won't follow Arch's libfprint updates by itself. Bump `pkgver`, update the checksum with `makepkg -g`, and rebuild with `makepkg -si`. If the patch doesn't apply anymore, it needs a refresh.

## How well it works

Tested with only this sensor. Most touches match and other fingers get rejected, and when a touch fails you just try again. Keep a password on your account though, I wouldn't trust a sensor this small as the only thing protecting anything that matters.

## Troubleshooting

**`Calibration failed!` during enroll.** Your finger was still on the sensor when the next touch started. Lift it all the way between touches.

**Verify never matches.** You probably enrolled before installing this. Run `fprintd-delete $USER` and enroll again.

**Verify fails some of the time.** That's normal for a sensor this small. If it fails most of the time, re-enroll and cover more of your finger.

**`swipe-too-short`.** You lifted your finger too fast, hold it for a second.

**`enroll-retry-scan`.** The driver threw away a bad capture, touch again.

**`makepkg` fails with `unknown public key`.** Your copy of the maintainer's key is old, fetch it again with the `gpg` line from [installing](#installing).

**`makepkg` fails with `No module named 'distutils'`.** Your `python3` is probably from pyenv, mise or asdf. Build with the system one:

```bash
PATH=/usr/bin:$PATH makepkg -si
```

For anything else, run fprintd with debug logs and run `fprintd-verify` in another terminal:

```bash
sudo systemctl stop fprintd
sudo G_MESSAGES_DEBUG=all /usr/lib/fprintd -t 2>&1 | grep -E 'elan|sigfm|error'
```

## Credits

SIGFM is by Matthieu Charette, Natasha England-Elbro and Timur Mangliev, the pure-C version here comes from [buxel/libfprint](https://github.com/buxel/libfprint).

The patch is LGPL-2.1-or-later like libfprint, the packaging files are 0BSD, see [LICENSE](LICENSE).

## Publishing

Pushing a tag like `v1.94.100-1` (matching `pkgver-pkgrel`) builds, tests and publishes to the AUR. It needs an `AUR_KEY` secret with an AUR SSH key. Regenerate `.SRCINFO` with `makepkg --printsrcinfo > .SRCINFO` after changing the PKGBUILD.
