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

It's on the AUR:

```bash
yay -S libfprint-elan-0c28-patch
```

Or build it from this repo, you need `base-devel`:

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

Enroll asks for a lot of touches. Hold still until it says `enroll-stage-passed`, then lift all the way, and move your finger around between touches so you cover more of it.

To use it for sudo, login, polkit prompts or your lock screen, see [docs/using-it.md](docs/using-it.md).

## Updating

This package won't follow Arch's libfprint updates by itself. When Arch ships a newer libfprint, use ***Flag package out-of-date*** on the [AUR page](https://aur.archlinux.org/packages/libfprint-elan-0c28-patch) and I'll patch and publish the new version, no need to build it yourself.

## How well it works

Tested with only this sensor. Most touches match and other fingers get rejected, and when a touch fails you just try again. Keep a password on your account though, I wouldn't trust a sensor this small as the only thing protecting anything that matters.

## Troubleshooting

**`enroll-unknown-error`, with `Device reported an error during enroll: Calibration failed!` in the fprintd log.** Your finger was still on the sensor when the next touch started. Lift it all the way between touches.

**`verify-no-match` every time, with `stored print has no SIGFM data, re-enroll this finger` in the log.** You enrolled before installing this. Run `fprintd-delete $USER` and enroll again.

**`verify-no-match` some of the time.** That's normal for a sensor this small. If it fails most of the time, re-enroll and cover more of your finger.

**`enroll-swipe-too-short` or `verify-swipe-too-short`.** You lifted your finger too fast, hold it for a second.

**`enroll-retry-scan`.** The driver threw away a bad capture, touch again.

**`enroll-remove-and-retry`.** Your finger landed before the sensor was ready for it, lift it and touch again.

**`makepkg` fails with `libfprint git repo ... FAILED (unknown public key 40F65066AD7C16DB)`.** Your copy of the maintainer's key is old, fetch it again with the `gpg` line from [installing](#installing).

**`makepkg` fails at `FAILED: libfprint/FPrint-2.0.gir` with `ModuleNotFoundError: No module named 'distutils'`.** Your `python3` is probably from pyenv, mise or asdf. Build with the system one:

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
