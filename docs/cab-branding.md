# Cabinet setup: boot branding

This page is optional. It replaces the generic Ubuntu graphics shown while the
cabinet boots with your own artwork. There are three separate things to brand,
each with its own mechanism.

| What you see | When | Mechanism |
|--------------|------|-----------|
| Boot splash | Between firmware and the login screen | Plymouth theme |
| Login background | The brief GDM greeter before autologin | GNOME Shell theme resource |
| Desktop wallpaper | After login, before an app loads | `gsettings` |

You need two images: a square logo for the boot splash (for example
`boot.png`), and a background image for the login screen and desktop (for
example `bg.png`).

## 1. Boot splash (Plymouth)

Create a Plymouth theme that shows your logo centered on black.

Create the directory `/usr/share/plymouth/themes/pinball/` and copy `boot.png`
into it. Add `pinball.plymouth`:

```ini
[Plymouth Theme]
Name=Pinball
Description=Custom pinball boot splash
ModuleName=script

[script]
ImageDir=/usr/share/plymouth/themes/pinball
ScriptFile=/usr/share/plymouth/themes/pinball/pinball.script
```

And `pinball.script`:

```
Window.SetBackgroundTopColor(0.0, 0.0, 0.0);
Window.SetBackgroundBottomColor(0.0, 0.0, 0.0);
logo.image  = Image("boot.png");
logo.sprite = Sprite(logo.image);
logo.sprite.SetX(Window.GetWidth()  / 2 - logo.image.GetWidth()  / 2);
logo.sprite.SetY(Window.GetHeight() / 2 - logo.image.GetHeight() / 2);
```

Activate the theme and rebuild the initramfs:

```sh
sudo update-alternatives --install /usr/share/plymouth/themes/default.plymouth \
    default.plymouth /usr/share/plymouth/themes/pinball/pinball.plymouth 200
sudo update-alternatives --set default.plymouth \
    /usr/share/plymouth/themes/pinball/pinball.plymouth
sudo update-initramfs -u
```

The splash appears on the next boot.

To revert:

```sh
sudo update-alternatives --set default.plymouth \
    /usr/share/plymouth/themes/bgrt/bgrt.plymouth
sudo update-initramfs -u
```

!!! note "Multi monitor splash"
    Plymouth draws on whichever output the kernel brings up first. On a multi
    monitor cabinet the splash may appear on a screen you did not expect. This
    is cosmetic and lasts only a few seconds.

## 2. Login background (GDM greeter)

On Ubuntu, the GDM greeter background is baked into a compiled GNOME Shell theme
resource. Changing it means editing that resource.

Install the GLib resource compiler if it is missing:

```sh
sudo apt-get install -y libglib2.0-dev-bin
```

Extract the current resource, patch the CSS, and recompile. The resource lives
at `/usr/share/gnome-shell/gnome-shell-theme.gresource`. Extract every entry,
add your `bg.png`, and append a rule to the greeter stylesheets:

```css
#lockDialogGroup {
    background: #000000 url("resource:///org/gnome/shell/theme/pinball-bg.png");
    background-size: cover;
    background-position: center;
    background-repeat: no-repeat;
}
```

Rebuild the resource with `glib-compile-resources`, back up the original to
`gnome-shell-theme.gresource.orig`, and install the new file in its place.
The change appears on the next GDM restart or reboot.

To revert, copy the `.orig` file back over the resource and restart GDM.

## 3. Desktop wallpaper

The wallpaper shown after login is a simple `gsettings` change. Copy `bg.png`
to a system path and point the background settings at it:

```sh
sudo install -m 644 bg.png /usr/share/backgrounds/pinball-bg.png
gsettings set org.gnome.desktop.background picture-uri \
    'file:///usr/share/backgrounds/pinball-bg.png'
gsettings set org.gnome.desktop.background picture-uri-dark \
    'file:///usr/share/backgrounds/pinball-bg.png'
gsettings set org.gnome.desktop.background picture-options 'zoom'
```

This takes effect immediately.

## Next

Continue with [Install Fliphetic](install.md).
