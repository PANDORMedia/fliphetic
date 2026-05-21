# Firmware ESP32

Une application Fliphetic peut flasher une ou plusieurs cartes ESP32 à chaque
chargement. Cette page explique comment la borne de flipper flashe le firmware,
et comment produire un binaire flashable à partir des chaînes d'outils
courantes : Arduino IDE, PlatformIO, ESP-IDF et MicroPython.

## Comment la borne flashe

Lorsqu'une application se charge, la borne exécute `esptool` pour chaque bloc
`[esp32.<name>]` du manifeste. La commande est, en pratique :

```sh
esptool --chip <chip> --port <port> --baud <baud> \
        write_flash <flash_addr> <firmware> [<extra_addr> <extra_file> ...]
```

* `<chip>`, `<port>` et `<baud>` proviennent de l'enregistrement du périphérique
  sur la borne et du manifeste.
* `<flash_addr>` et `<firmware>` proviennent du manifeste.
* `<extra_*>` sont des images supplémentaires facultatives, également issues du
  manifeste.

La borne ne compile pas le firmware. Vous le compilez une fois, vous committez
le binaire résultant dans votre dépôt, et la borne flashe ce fichier exact.

## Deux types d'image

Bien comprendre ceci est la clé d'un firmware fonctionnel.

### Image applicative

Une **image applicative** ne contient que votre programme. Elle doit être
flashée à l'offset applicatif, `0x10000` sur un partitionnement par défaut. Elle
ne fonctionne que si un bootloader et une table de partitions sont déjà présents
sur la puce.

### Image fusionnée

Une **image fusionnée** contient le bootloader, la table de partitions et
l'application dans un seul fichier. Elle est flashée à l'offset `0x0` et ne
dépend pas de ce qui se trouvait sur la puce auparavant. C'est le choix
recommandé pour Fliphetic, car un chargement est autonome et prévisible.

Vous pouvez toujours transformer un ensemble d'images séparées en une image
fusionnée avec `esptool` :

```sh
esptool --chip esp32 merge_bin -o merged.bin \
    0x1000  bootloader.bin \
    0x8000  partitions.bin \
    0x10000 app.bin
```

Le fichier de sortie `merged.bin` est ensuite flashé à `0x0`.

## Adresses de flash

| Type d'image | `flash_addr` dans le manifeste |
|------------|------------------------------|
| Image fusionnée (compilée avec `esptool merge_bin` ou exportée comme fusionnée) | `0x0` |
| Image applicative, ESP32 classique, partitions par défaut | `0x10000` |
| MicroPython précompilé, ESP32 classique | `0x1000` |
| MicroPython précompilé, ESP32-S3 / C3 / C6 | `0x0` |

En cas de doute, compilez une image fusionnée et utilisez `0x0`.

## Tutoriel : Arduino IDE

1. Installez le support de cartes ESP32. Dans Arduino IDE, ouvrez
   **File > Preferences**, ajoutez l'URL du gestionnaire de cartes Espressif,
   puis dans **Tools > Board > Boards Manager** installez
   **esp32 by Espressif Systems**.
2. Sélectionnez votre carte sous **Tools > Board**, ainsi que le bon port.
3. Écrivez et vérifiez votre sketch.
4. Choisissez **Sketch > Export Compiled Binary**. L'IDE écrit la sortie de
   compilation à côté de votre sketch, dans un sous-dossier `build/`.

Les versions récentes des cores ESP32 exportent plusieurs fichiers, dont un
`*.merged.bin`. Utilisez celui-ci et flashez-le à `0x0`.

Si votre core ne produit pas de fichier fusionné, il exporte l'application, le
bootloader et la table de partitions séparément. Fusionnez-les vous-même avec la
commande `esptool merge_bin` montrée ci-dessus, ou listez-les comme images
`extra` dans le manifeste.

## Tutoriel : PlatformIO

1. Installez l'extension [PlatformIO](https://platformio.org/) pour VS Code, ou
   le CLI PlatformIO Core.
2. Dans votre projet, compilez :

   ```sh
   pio run
   ```

3. L'image applicative est écrite dans
   `.pio/build/<env>/firmware.bin`. Le bootloader et la table de partitions se
   trouvent dans le même répertoire sous les noms `bootloader.bin` et
   `partitions.bin`.

Pour produire une image fusionnée, exécutez `esptool merge_bin` sur ces trois
fichiers comme montré ci-dessus, puis committez le binaire fusionné.

## Tutoriel : ESP-IDF

1. Installez [ESP-IDF](https://docs.espressif.com/projects/esp-idf/) et chargez
   son environnement.
2. Compilez :

   ```sh
   idf.py set-target esp32
   idf.py build
   ```

3. Les artefacts se trouvent dans `build/` : l'application dans
   `build/<project>.bin`, le bootloader dans
   `build/bootloader/bootloader.bin`, et la table de partitions dans
   `build/partition_table/partition-table.bin`.

ESP-IDF peut produire directement une image fusionnée :

```sh
idf.py merge-bin -o merged.bin
```

Committez `merged.bin` et flashez-le à `0x0`.

## Tutoriel : MicroPython

MicroPython présente une différence importante. Le binaire du firmware est
l'interpréteur MicroPython. Votre programme réel est un ensemble de fichiers
`.py` qui résident normalement sur le système de fichiers de la carte, et
Fliphetic ne **téléverse pas** de fichiers. Flasher un binaire MicroPython brut
vous donne donc un interpréteur nu sans programme.

Il y a deux façons d'utiliser MicroPython avec Fliphetic :

* **Code figé.** Compilez un firmware MicroPython personnalisé avec vos fichiers
  `.py` figés dans l'image (un manifeste au moment de la compilation). Le `.bin`
  résultant contient à la fois l'interpréteur et votre code, et se comporte
  comme n'importe quel autre firmware. C'est la voie recommandée.
* **Interpréteur précompilé.** Flashez le firmware précompilé depuis
  [micropython.org/download](https://micropython.org/download/) et acceptez que
  votre programme doive être placé sur la carte d'une autre manière. Cela ne
  s'accorde pas bien avec le modèle de Fliphetic.

Si vous flashez une image MicroPython précompilée, faites attention à l'adresse
de flash : l'ESP32 classique utilise `0x1000`, tandis que les ESP32-S3, C3 et C6
utilisent `0x0`.

Pour la plupart des projets de borne, Arduino IDE, PlatformIO ou ESP-IDF
conviennent mieux car ils produisent un binaire unique qui contient l'ensemble
du programme.

## Placer le binaire dans votre dépôt

Committez le binaire compilé dans le dépôt de votre application. Une disposition
courante :

```
firmware/
  src/                 your sources
  build/merged.bin     the binary you commit
  README.md            how to rebuild it
```

Ajoutez le reste de `build/` au `.gitignore`. Ne committez que le binaire final,
afin que la borne flashe un artefact reproductible et que les chargements
restent rapides.

## Bloc du manifeste

Référencez le binaire depuis `fliphetic.toml`. La clé après `esp32.` est le nom
du périphérique de la borne que vous ciblez, tel qu'enregistré sur la page
System du tableau de bord.

Une image fusionnée :

```toml
[esp32.buttons]
firmware   = "firmware/build/merged.bin"
chip       = "esp32"
flash_addr = "0x0"
required   = true
```

Une image applicative avec un bootloader et une table de partitions séparés :

```toml
[esp32.buttons]
firmware   = "firmware/build/app.bin"
chip       = "esp32"
flash_addr = "0x10000"
extra = [
  { addr = "0x1000", file = "firmware/build/bootloader.bin" },
  { addr = "0x8000", file = "firmware/build/partitions.bin" },
]
```

Référence des champs :

| Champ | Signification |
|-------|---------|
| `firmware` | Chemin du binaire principal, relatif à la racine du dépôt. |
| `chip` | Puce cible. Facultatif ; par défaut, la puce configurée pour le périphérique de la borne. |
| `baud` | Débit en bauds du flashage. Facultatif ; par défaut, le débit configuré pour le périphérique. |
| `flash_addr` | Offset auquel flasher `firmware`. Par défaut `0x10000`. |
| `extra` | Une liste d'images `{ addr, file }` flashées en même temps que l'image principale. |
| `required` | Si `false`, le chargement se poursuit lorsque le périphérique est absent ou que le flashage échoue. Par défaut `true`. |

Plusieurs cartes : ajoutez un bloc par périphérique, par exemple
`[esp32.buttons]` et `[esp32.lights]`.

Une application qui n'utilise aucun ESP32 omet simplement tous les blocs
`[esp32.*]`.

## Périphériques facultatifs

Définissez `required = false` lorsqu'une carte n'est pas essentielle. Si la
borne n'a aucun périphérique portant ce nom, ou que la carte n'est pas branchée,
le chargement consigne un avertissement et continue au lieu d'échouer. Les
périphériques requis manquants interrompent le chargement.

## Tester un firmware

Avant de vous fier à un chargement, vous pouvez flasher à la main sur la borne
pour confirmer que le binaire et les adresses sont corrects :

```sh
esptool --chip esp32 --port /dev/serial/by-id/usb-... --baud 921600 \
        write_flash 0x0 firmware/build/merged.bin
```

Si cela réussit, les mêmes paramètres dans votre manifeste réussiront aussi.

## Le protocole des boutons

Fliphetic flashe le firmware mais ne définit pas comment le firmware communique
avec votre application. Le protocole (sur le port série USB, sur le réseau, sous
forme d'appuis de touches émulés, ou tout autre moyen) est décidé entre votre
firmware et votre application. Documentez-le dans votre dépôt afin que les deux
moitiés restent synchronisées.
