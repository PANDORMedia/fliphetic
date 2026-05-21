# Configuration de la borne : habillage au démarrage

Cette page est facultative. Elle remplace les graphismes Ubuntu génériques affichés pendant le démarrage de la borne de flipper par vos propres visuels. Il y a trois éléments distincts à personnaliser, chacun avec son propre mécanisme.

| Ce que vous voyez | Quand | Mécanisme |
|--------------|------|-----------|
| Écran de démarrage | Entre le firmware et l'écran de connexion | Thème Plymouth |
| Arrière-plan de connexion | Le bref greeter GDM avant la connexion automatique | Ressource de thème GNOME Shell |
| Fond d'écran du bureau | Après la connexion, avant le chargement d'une application | `gsettings` |

Vous avez besoin de deux images : un logo carré pour l'écran de démarrage (par exemple `boot.png`), et une image d'arrière-plan pour l'écran de connexion et le bureau (par exemple `bg.png`).

## 1. Écran de démarrage (Plymouth)

Créez un thème Plymouth qui affiche votre logo centré sur fond noir.

Créez le répertoire `/usr/share/plymouth/themes/pinball/` et copiez-y `boot.png`. Ajoutez `pinball.plymouth` :

```ini
[Plymouth Theme]
Name=Pinball
Description=Custom pinball boot splash
ModuleName=script

[script]
ImageDir=/usr/share/plymouth/themes/pinball
ScriptFile=/usr/share/plymouth/themes/pinball/pinball.script
```

Et `pinball.script` :

```
Window.SetBackgroundTopColor(0.0, 0.0, 0.0);
Window.SetBackgroundBottomColor(0.0, 0.0, 0.0);
logo.image  = Image("boot.png");
logo.sprite = Sprite(logo.image);
logo.sprite.SetX(Window.GetWidth()  / 2 - logo.image.GetWidth()  / 2);
logo.sprite.SetY(Window.GetHeight() / 2 - logo.image.GetHeight() / 2);
```

Activez le thème et reconstruisez l'initramfs :

```sh
sudo update-alternatives --install /usr/share/plymouth/themes/default.plymouth \
    default.plymouth /usr/share/plymouth/themes/pinball/pinball.plymouth 200
sudo update-alternatives --set default.plymouth \
    /usr/share/plymouth/themes/pinball/pinball.plymouth
sudo update-initramfs -u
```

L'écran de démarrage apparaît au prochain démarrage.

Pour revenir en arrière :

```sh
sudo update-alternatives --set default.plymouth \
    /usr/share/plymouth/themes/bgrt/bgrt.plymouth
sudo update-initramfs -u
```

!!! note "Écran de démarrage multi-moniteur"
    Plymouth dessine sur la sortie que le noyau active en premier. Sur une borne multi-moniteur, l'écran de démarrage peut apparaître sur un écran auquel vous ne vous attendiez pas. C'est purement cosmétique et cela ne dure que quelques secondes.

## 2. Arrière-plan de connexion (greeter GDM)

Sur Ubuntu, l'arrière-plan du greeter GDM est intégré dans une ressource de thème GNOME Shell compilée. Le modifier implique de modifier cette ressource.

Installez le compilateur de ressources GLib s'il est absent :

```sh
sudo apt-get install -y libglib2.0-dev-bin
```

Extrayez la ressource actuelle, corrigez le CSS, puis recompilez. La ressource se trouve dans `/usr/share/gnome-shell/gnome-shell-theme.gresource`. Extrayez chaque entrée, ajoutez votre `bg.png`, et ajoutez une règle aux feuilles de style du greeter :

```css
#lockDialogGroup {
    background: #000000 url("resource:///org/gnome/shell/theme/pinball-bg.png");
    background-size: cover;
    background-position: center;
    background-repeat: no-repeat;
}
```

Reconstruisez la ressource avec `glib-compile-resources`, sauvegardez l'original sous `gnome-shell-theme.gresource.orig`, et installez le nouveau fichier à sa place. Le changement apparaît au prochain redémarrage de GDM ou au prochain redémarrage du système.

Pour revenir en arrière, recopiez le fichier `.orig` sur la ressource et redémarrez GDM.

## 3. Fond d'écran du bureau

Le fond d'écran affiché après la connexion est une simple modification via `gsettings`. Copiez `bg.png` vers un chemin système et faites pointer les paramètres d'arrière-plan dessus :

```sh
sudo install -m 644 bg.png /usr/share/backgrounds/pinball-bg.png
gsettings set org.gnome.desktop.background picture-uri \
    'file:///usr/share/backgrounds/pinball-bg.png'
gsettings set org.gnome.desktop.background picture-uri-dark \
    'file:///usr/share/backgrounds/pinball-bg.png'
gsettings set org.gnome.desktop.background picture-options 'zoom'
```

Cela prend effet immédiatement.

## Suite

Poursuivez avec [Installer Fliphetic](install.md).
