# Point d'accès Wi-Fi

Une borne Fliphetic peut diffuser son propre réseau Wi-Fi. Les cartes ESP32,
les téléphones et les ordinateurs portables rejoignent ce réseau pour atteindre
le tableau de bord et internet, sans dépendre du Wi-Fi du lieu.

C'est optionnel. Une borne fonctionne très bien avec sa seule connexion
filaire. Configurez le point d'accès lorsque vous avez des cartes ESP32 (ou
d'autres appareils) qui doivent communiquer avec la borne par Wi-Fi.

## Fonctionnement

La borne héberge le point d'accès grâce à un **adaptateur Wi-Fi USB**.
NetworkManager fait le travail en mode *partagé* (« shared ») :

* il distribue des adresses par DHCP — chaque appareil qui se connecte reçoit
  une IP ;
* il achemine (« NAT ») le trafic du réseau Wi-Fi vers la liaison filaire de la
  borne, de sorte que les appareils connectés ont aussi accès à internet.

Le réseau Wi-Fi utilise la plage `10.42.0.0/24`, et la borne elle-même est
toujours joignable à l'adresse de passerelle **`10.42.0.1`**.

!!! warning "Conservez la connexion filaire"
    Le point d'accès partage la liaison filaire de la borne — laissez le câble
    Ethernet branché. C'est aussi votre filet de sécurité : vous gérez la borne
    par le réseau local filaire et par Tailscale, donc un adaptateur Wi-Fi mal
    configuré ne peut jamais vous bloquer l'accès au tableau de bord.

## Choisir un adaptateur Wi-Fi USB

Tous les adaptateurs Wi-Fi USB ne peuvent pas fonctionner en point d'accès —
beaucoup ne fonctionnent qu'en client. Choisissez-en un dont le pilote Linux
prend en charge le **mode AP** (point d'accès).

Les adaptateurs basés sur le pilote MediaTek **mt76** (par exemple le MT7612U
ou le MT7921AU) prennent en charge le mode AP et fonctionnent sur Ubuntu 24.04
sans configuration. C'est un choix sûr.

!!! tip "Vérifiez avant de compter dessus"
    L'adaptateur branché, lancez `iw list` et cherchez `AP` sous *Supported
    interface modes*. Si `AP` est listé, l'adaptateur peut héberger le point
    d'accès. Si l'adaptateur nécessite un pilote hors-arbre, installez et
    vérifiez ce pilote avant d'en dépendre.

## Configurer le point d'accès

Le point d'accès se configure depuis le tableau de bord, sur la page
**Network** (réseau).

1. Branchez l'adaptateur Wi-Fi USB sur la borne.
2. Ouvrez le tableau de bord et allez sur **Network**. La page liste chaque
   interface Wi-Fi détectée. Si elle n'en affiche aucune, l'adaptateur n'est
   pas reconnu — voir la section ci-dessus.
3. Sous **Access point** (point d'accès), renseignez :
    * **Wi-Fi interface** — l'adaptateur détecté.
    * **SSID** — le nom du réseau. Un nom suggéré est pré-rempli ; modifiez-le
      si vous le souhaitez.
    * **WPA2 passphrase** — de 8 à 63 caractères. C'est le mot de passe que les
      appareils utilisent pour se connecter. Choisissez-en un vrai.
4. Sélectionnez **Create AP**. La borne commence à diffuser immédiatement.

Une fois créé, la page Network affiche le SSID, l'interface, la passerelle et
l'état du point d'accès, avec les boutons **Start AP** et **Stop AP**. Le point
d'accès se reconnecte automatiquement au redémarrage de la borne.

## Appareils connectés et adresses fixes

La page Network liste chaque appareil actuellement connecté au point d'accès —
son IP, son adresse MAC, son nom d'hôte et la durée restante de son bail.

Par défaut, chaque appareil reçoit l'adresse que le DHCP lui attribue, et cette
adresse peut changer. Pour un appareil que vous voulez joindre à une adresse
prévisible — typiquement un ESP32 — fixez-le :

* sur une ligne sous **Connected devices**, sélectionnez **Reserve IP** ; ou
* sous **DHCP reservations**, choisissez **Add reservation** et saisissez
  l'adresse MAC de l'appareil ainsi que l'IP à lui attribuer.

Une réservation lie une adresse MAC à une IP fixe dans la plage `10.42.0.0/24`,
et prend effet immédiatement.

## Recette : connecter un ESP32 au Wi-Fi de la borne

Un ESP32 rejoint le point d'accès de la borne exactement comme il rejoindrait
n'importe quel réseau Wi-Fi. Avec le framework Arduino :

```cpp
#include <WiFi.h>

// Gardez ces valeurs hors du dépôt — voir l'avertissement ci-dessous.
#ifndef WIFI_SSID
#define WIFI_SSID "YOUR_CAB_SSID"
#endif
#ifndef WIFI_PASS
#define WIFI_PASS "YOUR_WIFI_PASSPHRASE"
#endif

void setup() {
  Serial.begin(115200);
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    // connecté — WiFi.localIP() est l'adresse attribuée par le DHCP de la borne
  } else {
    WiFi.begin(WIFI_SSID, WIFI_PASS);   // on continue d'essayer
    delay(2000);
  }
}
```

Une fois associée, la carte peut atteindre :

* **internet** — le point d'accès fait du NAT vers la liaison de la borne ;
* **la borne** — à la passerelle, `10.42.0.1` ;
* **un service de la borne** — à `10.42.0.1:<port>`, *si* ce service publie un
  port hôte **fixe**. Un service Compose qui publie `["8099:80"]` est joignable
  depuis l'ESP32 à `10.42.0.1:8099`. Un service qui publie un port éphémère
  (`["0:80"]`) reçoit un port différent à chaque chargement et ne peut pas être
  ciblé depuis le firmware.

!!! danger "Ne committez jamais d'identifiants Wi-Fi"
    Un dépôt d'application est généralement public. Ne codez pas en dur la
    passphrase dans un fichier source que vous committez. Passez `WIFI_SSID` et
    `WIFI_PASS` comme indicateurs de compilation (dans `platformio.ini`, ou via
    un secret CI), ou gardez-les dans un petit en-tête listé dans
    `.gitignore`. Les valeurs de repli `#ifndef` ci-dessus ne servent qu'à
    laisser une compilation nue aboutir.

La forme par indicateur de compilation dans `platformio.ini` :

```ini
build_flags =
  -DWIFI_SSID='"YOUR_CAB_SSID"'
  -DWIFI_PASS='"YOUR_WIFI_PASSPHRASE"'
```

### Fixer la carte à une IP fixe

Si votre application doit atteindre l'ESP32 (et pas seulement l'inverse),
attribuez à la carte une réservation DHCP comme décrit ci-dessus. Lisez une
fois l'adresse MAC de la carte — elle apparaît dans la liste des appareils
connectés, et `WiFi.macAddress()` l'affiche — puis réservez-lui une adresse sur
la page Network.

## Dépannage

| Symptôme | Cause probable |
|----------|----------------|
| La page Network n'affiche aucune interface Wi-Fi | Adaptateur non reconnu, ou pilote manquant. Vérifiez avec `iw list`. |
| « Create AP » échoue | L'adaptateur ne prend pas en charge le mode AP. Confirmez que `AP` apparaît dans `iw list`. |
| Les appareils se connectent mais n'ont pas internet | La liaison filaire de la borne est coupée — le point d'accès fait son NAT par celle-ci. |
| L'ESP32 n'atteint jamais `WL_CONNECTED` | SSID ou passphrase incorrects, ou point d'accès arrêté. Vérifiez l'état sur la page Network. |
| L'ESP32 se connecte mais n'atteint pas un service | Le service publie un port éphémère. Attribuez-lui un port hôte fixe. |
