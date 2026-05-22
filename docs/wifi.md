# Wi-Fi access point

A Fliphetic cabinet can broadcast its own Wi-Fi network. ESP32 boards, phones,
and laptops join that network to reach the dashboard and the internet, without
depending on the venue's Wi-Fi.

This is optional. A cabinet works fine with only its wired connection. Set up
the access point when you have ESP32 boards (or other devices) that need to
talk to the cabinet over Wi-Fi.

## How it works

The cabinet runs the access point off a **USB Wi-Fi adapter**. NetworkManager
does the work in *shared* mode:

* it hands out addresses over DHCP — every device that joins gets an IP;
* it routes ("NATs") traffic from the Wi-Fi network out through the cabinet's
  wired uplink, so joined devices also have internet access.

The Wi-Fi network uses the `10.42.0.0/24` range, and the cabinet itself is
always reachable at the gateway address **`10.42.0.1`**.

!!! warning "Keep the wired connection"
    The access point shares the cabinet's wired uplink — leave the Ethernet
    cable plugged in. It is also your safety net: you manage the cabinet over
    the wired LAN and Tailscale, so a misconfigured Wi-Fi adapter can never
    lock you out of the dashboard.

## Choosing a USB Wi-Fi adapter

Not every USB Wi-Fi adapter can run as an access point — many work only as
clients. Pick one whose Linux driver supports **AP mode**.

Adapters based on the MediaTek **mt76** driver (for example the MT7612U or the
MT7921AU) support AP mode and work on Ubuntu 24.04 out of the box. They are a
safe choice.

!!! tip "Check before you rely on it"
    With the adapter plugged in, run `iw list` and look under *Supported
    interface modes* for `AP`. If `AP` is listed, the adapter can host the
    access point. If the adapter needs an out-of-tree driver, install and
    verify that driver before depending on it.

## Setting up the access point

The access point is configured from the dashboard, on the **Network** page.

1. Plug the USB Wi-Fi adapter into the cabinet.
2. Open the dashboard and go to **Network**. The page lists every Wi-Fi
   interface it detects. If it shows none, the adapter is not recognised — see
   the section above.
3. Under **Access point**, fill in:
    * **Wi-Fi interface** — the detected adapter.
    * **SSID** — the network name. A suggested name is pre-filled; change it if
      you like.
    * **WPA2 passphrase** — 8 to 63 characters. This is the password devices
      use to join. Choose a real one.
4. Select **Create AP**. The cabinet starts broadcasting immediately.

Once created, the Network page shows the access point's SSID, interface,
gateway, and state, with **Start AP** and **Stop AP** controls. The access
point reconnects automatically when the cabinet reboots.

## Connected devices and fixed addresses

The Network page lists every device currently joined to the access point —
its IP, MAC address, hostname, and remaining lease time.

By default each device gets whatever address DHCP hands out, and that address
can change. For a device you want to reach at a predictable address — typically
an ESP32 — pin it:

* on a row under **Connected devices**, select **Reserve IP**; or
* under **DHCP reservations**, choose **Add reservation** and enter the
  device's MAC address and the IP to assign it.

A reservation ties a MAC address to a fixed IP in the `10.42.0.0/24` range, and
takes effect immediately.

## Recipe: connecting an ESP32 to the cabinet Wi-Fi

An ESP32 joins the cabinet's access point exactly as it would join any Wi-Fi
network. With the Arduino framework:

```cpp
#include <WiFi.h>

// Keep these out of source control — see the warning below.
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
    // joined — WiFi.localIP() is the address the cabinet's DHCP assigned
  } else {
    WiFi.begin(WIFI_SSID, WIFI_PASS);   // keep retrying
    delay(2000);
  }
}
```

Once associated, the board can reach:

* **the internet** — the access point NATs out through the cabinet's uplink;
* **the cabinet** — at the gateway, `10.42.0.1`;
* **a cabinet service** — at `10.42.0.1:<port>`, *if* that service publishes a
  **fixed** host port. A Compose service that publishes `["8099:80"]` is
  reachable from the ESP32 at `10.42.0.1:8099`. A service that publishes an
  ephemeral port (`["0:80"]`) gets a different port on every load and cannot be
  targeted from firmware.

!!! danger "Never commit Wi-Fi credentials"
    An app repository is usually public. Do not hard-code the passphrase in a
    source file you commit. Pass `WIFI_SSID` and `WIFI_PASS` as build flags
    (in `platformio.ini`, or as a CI secret), or keep them in a small header
    listed in `.gitignore`. The `#ifndef` fallbacks above only keep a bare
    build compiling.

The build-flag form in `platformio.ini`:

```ini
build_flags =
  -DWIFI_SSID='"YOUR_CAB_SSID"'
  -DWIFI_PASS='"YOUR_WIFI_PASSPHRASE"'
```

### Pin the board to a fixed IP

If your app needs to reach the ESP32 (not just the other way round), give the
board a DHCP reservation as described above. Read the board's MAC address once
— it appears on the connected-devices list, and `WiFi.macAddress()` prints it
— then reserve an address for it on the Network page.

## Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| Network page shows no Wi-Fi interface | Adapter not recognised, or its driver is missing. Check `iw list`. |
| "Create AP" fails | The adapter does not support AP mode. Confirm `AP` appears in `iw list`. |
| Devices join but have no internet | The cabinet's wired uplink is down — the access point NATs through it. |
| ESP32 never reaches `WL_CONNECTED` | Wrong SSID or passphrase, or the access point is stopped. Check the Network page state. |
| ESP32 connects but can't reach a service | The service publishes an ephemeral port. Give it a fixed host port. |
