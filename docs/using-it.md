# Using it for sudo, login and the rest

Once a finger is enrolled, fprintd works, but nothing asks for your fingerprint until you tell PAM to. On Arch you do that per service, by adding one line to the top of the `auth` section of its file in `/etc/pam.d/`:

```
auth      sufficient   pam_fprintd.so
```

`sufficient` means a matching finger is enough, and if it fails or times out PAM moves on to your password like before. Always edit these files with `sudoedit` and keep a root shell open in another terminal until you've tested it, a typo in a PAM file can lock you out of `sudo`.

> [!NOTE]
> This sensor misses some touches, so `pam_fprintd` giving you 3 tries by default helps a lot. You can change that with `max-tries=` and `timeout=` (seconds), for example `pam_fprintd.so max-tries=5 timeout=20`.

## sudo

```bash
sudoedit /etc/pam.d/sudo
```

Put the line right above `auth include system-auth`, so the top of the file looks like this:

```
#%PAM-1.0
auth      sufficient   pam_fprintd.so
auth      include      system-auth
```

Open a new terminal and run `sudo true`, it should say `Place your finger on the fingerprint reader`.

## Polkit prompts

The password dialogs from GUI apps (mounting disks, `pkexec`, system settings) go through `polkit-1`. Arch ships its file in `/usr/lib/pam.d/`, and a copy in `/etc/pam.d/` overrides it, so copy it first:

```bash
sudo cp /usr/lib/pam.d/polkit-1 /etc/pam.d/polkit-1
sudoedit /etc/pam.d/polkit-1
```

Add the same `auth sufficient pam_fprintd.so` line above the first `auth` line.

## TTY login

```bash
sudoedit /etc/pam.d/system-local-login
```

Same line, at the top of the `auth` section. This covers logging in on a TTY and display managers that include `system-local-login`.

## Desktop environments

GNOME and KDE don't need any PAM editing, enroll from their settings and turn it on there:

- **GNOME**: ***Settings -> System -> Users -> Fingerprint Login***. GDM uses it at the login screen.
- **KDE Plasma**: ***System Settings -> Users -> Configure Fingerprint Authentication***. SDDM and the lock screen use it.

Fingerprint login can't unlock your keyring (GNOME Keyring or KWallet), because those need your password to decrypt. If you log in with your finger, you'll get asked for the keyring password the first time an app needs it.

## Hyprland (hyprlock)

hyprlock talks to fprintd by itself, add this to `~/.config/hypr/hyprlock.conf`:

```
auth {
    fingerprint {
        enabled = true
    }
}
```

## Password and fingerprint at the same time

With `pam_fprintd` you have to wait for the fingerprint prompt to give up (or use up its tries) before you can type your password. If that bothers you, `pam-fprint-grosshack-fork-git` on the AUR accepts either one at the same prompt. Instead of the `pam_fprintd.so` line, add these two:

```
auth      sufficient   pam_fprintd_grosshack.so quiet prompt=[pwd or fp:]
auth      sufficient   pam_unix.so try_first_pass nullok
```

## More fingers

`fprintd-enroll` enrolls your right index finger by default. To add others:

```bash
fprintd-enroll -f left-index-finger
fprintd-list $USER
```

## Turning it off

Remove the `pam_fprintd.so` line from the files you edited. Your enrolled prints stay in `/var/lib/fprint/` until you run `fprintd-delete $USER`.
