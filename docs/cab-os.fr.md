# Configuration de la borne : système d'exploitation et réseau

Cette page met en place le système d'exploitation et l'accès distant. À la fin,
vous disposez d'une borne sous Ubuntu qui démarre directement sur un bureau et
qui est accessible par SSH via Tailscale.

## 1. Installer Ubuntu 24.04

Installez Ubuntu 24.04 LTS sur le PC de la borne. Pendant ou après
l'installation :

* Créez un utilisateur nommé `flipper`.
* Donnez à `flipper` les droits d'administrateur (sudo).
* Choisissez l'installation de bureau normale pour que GNOME et GDM soient
  présents.

Après le premier démarrage, assurez-vous que la session graphique est **Xorg**,
pas Wayland. À l'écran de connexion GDM, cliquez sur l'icône en forme
d'engrenage et choisissez « Ubuntu sur Xorg ». Fliphetic place les fenêtres de
kiosque avec des outils X11 et a besoin de la session Xorg.

## 2. Activer la connexion automatique

Une borne doit démarrer sur son bureau sans interaction au clavier. Activez la
connexion automatique GDM pour `flipper`.

Modifiez `/etc/gdm3/custom.conf` et définissez, sous `[daemon]` :

```ini
AutomaticLoginEnable=true
AutomaticLogin=flipper
```

Redémarrez et confirmez que la borne arrive sur le bureau sans demande de mot de
passe.

## 3. Installer Tailscale

Tailscale donne à la borne une adresse privée stable que vous et vos étudiants
pouvez atteindre depuis n'importe où, sans ouvrir de ports.

```sh
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Suivez l'URL affichée pour authentifier la borne dans votre tailnet. Lisez
ensuite l'adresse IPv4 Tailscale de la borne :

```sh
tailscale ip -4
```

Notez cette adresse. Elle est utilisée partout : le tableau de bord y écoute, et
le script de déploiement la cible. Dans cette documentation, le marqueur
`100.x.y.z` représente l'adresse Tailscale de votre borne.

!!! note "Les adresses Tailscale sont stables"
    Un nœud Tailscale conserve son IP pendant toute la durée de vie du nœud,
    vous avez donc rarement besoin de la changer. Si elle venait à changer,
    mettez à jour `bind_host` dans `/etc/fliphetic/config.toml` ainsi que la
    cible dans le script de déploiement.

## 4. Configurer l'accès SSH depuis votre poste de travail

Vous gérez et déployez la borne depuis un poste de travail séparé par SSH.
Installez une clé pour ne pas avoir à taper de mots de passe.

Depuis votre poste de travail, avec une clé SSH existante (ou après
`ssh-keygen`) :

```sh
ssh-copy-id flipper@100.x.y.z
```

Confirmez que la connexion par clé fonctionne :

```sh
ssh flipper@100.x.y.z 'echo connected as $(whoami)'
```

Vous pouvez éventuellement ajouter un alias d'hôte à `~/.ssh/config` sur votre
poste de travail :

```
Host flipper
    HostName 100.x.y.z
    User flipper
```

Après cela, `ssh flipper` suffit.

## 5. Maintenir la borne sur le réseau

La borne a besoin :

* d'un accès internet, pour les téléchargements d'images Docker et le clonage
  des dépôts étudiants ;
* d'une connectivité Tailscale, pour le tableau de bord.

Si la borne utilise le Wi-Fi, assurez-vous que la connexion est configurée pour
se connecter automatiquement et qu'elle est disponible pour la session
`flipper` au démarrage.

## Suite

Continuez avec [Affichage et alimentation](cab-display-power.md) pour configurer
les trois écrans et le comportement à la mise sous tension.
