# AC Infinity HACS Integration (AirTap Fork)

Fork of [hunterjm/ac-infinity-hacs](https://github.com/hunterjm/ac-infinity-hacs) with added support for **AC Infinity AirTap** devices (T4, T6, etc.) over Bluetooth.

## What This Fork Changes

- Adds AirTap (device type 6) to supported device models
- Discovers AirTap devices by BLE local name (`BLE_FAN`) since they don't advertise manufacturer data
- Adds `TURN_ON`/`TURN_OFF` fan feature flags for newer Home Assistant versions
- Adds VPD sensor support for AirTap devices

## Known Limitations

- **Humidity and VPD sensors** show 0 -- the AirTap does not report these values over BLE
- **Temperature** works correctly
- **Fan control** (on/off and speed 0-100% in 10% increments) works
- **Weak BLE signal** (RSSI below -80) can cause intermittent polling errors -- move your Bluetooth proxy closer to the device
- **Close the AC Infinity phone app** before setup -- BLE only allows one connection at a time

## Installation (Podman / Rootless Container)

These steps assume Home Assistant is running in a rootless Podman container (Quadlet) as user 1000.

### 1. Install via HACS

1. In Home Assistant, go to **HACS** > **Integrations** > **3 dots menu** > **Custom repositories**
2. Add this repo URL and select **Integration** as the category
3. Install the **AC Infinity** integration from HACS
4. **Do NOT restart yet** -- the pip dependency install will fail due to container permissions

### 2. Install the `ac-infinity-ble` dependency

The pip install fails inside the container because `/usr/local/lib/python3.13/site-packages/` is not writable by user 1000. Install manually:

```bash
# Stop the container
systemctl --user stop homeassistant-ui

# Install the dependency into /config/deps (persisted on volume)
podman run --rm -it \
  --user root \
  -v homeassistant-data:/config:z \
  ghcr.io/home-assistant/home-assistant:stable \
  sh -c "mkdir -p /config/deps && pip install --target /config/deps ac-infinity-ble==0.4.3"
```

### 3. Clean up conflicting dependencies

The pip install pulls in old versions of BLE libraries that conflict with HA's built-in versions. Remove everything except `ac_infinity_ble`:

```bash
podman run --rm -it \
  --user root \
  -v homeassistant-data:/config:z \
  ghcr.io/home-assistant/home-assistant:stable \
  sh -c "cd /config/deps && rm -rf bleak bleak-*.dist-info bleak_retry_connector-*.dist-info bluetooth_adapters bluetooth_adapters-*.dist-info dbus_fast dbus_fast-*.dist-info aiooui aiooui-*.dist-info uart_devices uart_devices-*.dist-info usb_devices usb_devices-*.dist-info"
```

### 4. Patch the `ac-infinity-ble` library for AirTap

The library expects BLE response data sized for CLOUDLINE fans. AirTap returns shorter data, causing an `IndexError`. Apply this patch:

```bash
podman run --rm -it \
  --user root \
  -v homeassistant-data:/config:z \
  ghcr.io/home-assistant/home-assistant:stable \
  sed -i \
    -e 's/self\._state\.work_type = data\[12\]/self._state.work_type = data[12] if len(data) > 12 else self._state.work_type/' \
    -e 's/self\._state\.level_off = data\[15\]/self._state.level_off = data[15] if len(data) > 15 else self._state.level_off/' \
    -e 's/self\._state\.level_on = data\[18\]/self._state.level_on = data[18] if len(data) > 18 else self._state.level_on/' \
    /config/deps/ac_infinity_ble/device.py
```

### 5. Start the container

```bash
systemctl --user start homeassistant-ui
```

### 6. Add the integration

1. Go to **Settings** > **Devices & Services** > **Add Integration** > search **AC Infinity**
2. **Close the AC Infinity phone app** first so BLE is free
3. Select your AirTap device (`BLE_FAN`) from the list

## Automating the Patch (Quadlet)

To avoid re-running the `sed` patch manually after every container restart, add this to your Quadlet `.container` file under `[Service]`:

```ini
ExecStartPost=podman exec --user root homeassistant-ui sed -i -e 's/self\._state\.work_type = data\[12\]/self._state.work_type = data[12] if len(data) > 12 else self._state.work_type/' -e 's/self\._state\.level_off = data\[15\]/self._state.level_off = data[15] if len(data) > 15 else self._state.level_off/' -e 's/self\._state\.level_on = data\[18\]/self._state.level_on = data[18] if len(data) > 18 else self._state.level_on/' /config/deps/ac_infinity_ble/device.py
```

Then reload:

```bash
systemctl --user daemon-reload
```

## After a New HA Image / Reinstall

If you pull a new HA container image or reinstall, repeat steps **2 through 5**. The HACS integration files persist on the volume (`/config/custom_components/`), but the pip dependency and patch may need to be reapplied.
