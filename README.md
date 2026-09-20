# kauf-plugs

Unified ESPHome package repo for KAUF smart plugs (PLF10 and PLF12).

The recommended way to import a plug into your ESPHome dashboard is through the dashboard import feature. The plug should show up in the ESPHome dashboard as being available to adopt. Below is the minimum necessary yaml, which can be used to manually add your device instead. Change the `name` and `friendly_name` substitutions to fit your use case. Adding a [`use_address:`](https://esphome.io/components/wifi.html?highlight=use_address#configuration-variables) under `wifi:` will allow you to point the ESPHome dashboard to the device on your network for flashing OTA.

The `friendly_name` substitution is recommended and will not be automatically created by the ESPHome dashboard import. If you use the ESPHome dashboard import feature, we recommend that you add a `friendly_name` substitution to rename all of the entities in Home Assistant in one line of yaml.

## Current Products

- Current sold model: **PLF12** — manual: [`PLF12 Manual.pdf`](./PLF12%20Manual.pdf)
- **PLF10** — earlier hardware revision, still supported here. (Manual not yet copied into this repo; see [`KaufHA/PLF10`](https://github.com/KaufHA/PLF10).)

## Quick Start

```yaml
substitutions:
  name: bed-plug
  friendly_name: Bed Plug
  component_family: kauf      # kauf | stock
  component_source: release   # release | beta | local (kauf only)
  profile: default            # minimal | lite | default

packages:
  Kauf.Plug: github://KaufHA/kauf-plugs/packages/kauf-plf12.yaml

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
```

## Entrypoints

- `packages/kauf-plf10.yaml`: PLF10 entrypoint. Defaults to the `kauf` family, pinned `release` source, and `default` profile.
- `packages/kauf-plf10-lite.yaml`: PLF10 convenience entrypoint using the `kauf` family, pinned `release` source, and `lite` profile.
- `packages/kauf-plf12.yaml`: PLF12 entrypoint. Defaults to the `kauf` family, pinned `release` source, and `default` profile.
- `packages/kauf-plf12-lite.yaml`: PLF12 convenience entrypoint using the `kauf` family, pinned `release` source, and `lite` profile.
- `packages/kauf-plug.yaml`: Generic entrypoint with no defaults. Requires `component_family`, `component_source`, `profile`, and `target` substitutions.

Each hardware-specific entrypoint declares `component_family`/`component_source`/`profile` substitutions (overridable in your own YAML) and composes:

- `profiles/${component_family}-${profile}.yaml`
- `targets/${component_family}-<model>-target.yaml`

## Package Layout

- `packages/profiles/`: `stock-lite.yaml` is the concrete baseline. Both Default profiles inherit their corresponding Lite profile and add the shared Plus setting entities from `default-additions.yaml`; `kauf-default.yaml` also stamps the additional persistent entities. `minimal` remains a separate stripped-down profile.
- `packages/targets/`: hardware/model overrides, split into `${component_family}-<model>-target.yaml` (family-specific board tweaks such as `start_free`) and `<model>-common.yaml` (shared pins, calibration, and board configuration)
- `packages/power/`: two power-monitoring backends: ESPHome's stock `hlw8012` implementation and Kauf's `kauf_hlw8012` implementation. Profile-specific scaling and threshold behavior lives in the profile layer.
- `packages/common-source/`: selects the `release`, `beta`, or `local` build of [`KaufHA/common`](https://github.com/KaufHA/common) for the Kauf component family. Stock does not load external components.
- `packages/shared/`: cross-cutting includes (e.g. versioning)
- `packages/updates/`: Yaml files used to build OTA update and factory-test binaries. Generally not useful to end users.

## Include Tree

Example: running `packages/kauf-plf12.yaml` (`component_family: kauf`, `component_source: release`, `profile: default`) resolves as:

```
kauf-plf12.yaml                              (family=kauf, source=release, profile=default)
├─ Kauf.Profile → profiles/kauf-default.yaml
│   ├─ Kauf.Lite → profiles/kauf-lite.yaml
│   │   ├─ stock → profiles/stock-lite.yaml
│   │   │   ├─ versioning → shared/kauf-versioning.yaml
│   │   │   └─ sensor include → power/${component_family}-hlw8012.yaml
│   │   ├─ Kauf.Common → common-source/${component_source}.yaml
│   │   └─ Kauf-specific entities and core flash stamping
│   ├─ Default.Additions → profiles/default-additions.yaml
│   └─ inline flash metadata for the additional Default entities
└─ Kauf.Target → targets/kauf-plf12-target.yaml
    └─ common_target → targets/plf12-common.yaml (GPIO pins, calibration, board)
```

Three things worth knowing when reading this:

- **`component_family` and `profile` pick which profile and target files load** through `${component_family}-${profile}` and `${component_family}-<model>-target` path interpolation. The stock family swaps the Kauf layers for stock ESPHome implementations.
- **`component_source` only selects where the Kauf external components come from**: a pinned release, the latest default branch, or a sibling local checkout. It does not require duplicate profile or target files.
- **Each concrete stock profile includes the family-selected power sensor definition** (`kauf-hlw8012.yaml` or `stock-hlw8012.yaml`). Targets supply the model-specific pins and calibration values.

## Profile Options

Available for both `kauf` and `stock` component families. `minimal`, `lite`, and `default` select different entity scopes.

- **`minimal`** — Bare-bones smart plug. Button toggles the relay; the blue LED automatically follows relay state and the red LED blinks as a Wi-Fi/Home-Assistant connection-status indicator, but neither is exposed as a controllable entity. Power monitoring (Power/Current/Voltage + Total Daily Energy) is reported with fixed, non-configurable settings. No LED, debounce, threshold, or update-interval configuration, and no diagnostics entities.
- **`lite`** — Former base/default behavior. Keeps the full control, power-monitoring, and diagnostic entity set, but omits the Plus setting entities (No HASS, Debounce Time, Use Threshold, Monitoring Update Interval, Boot State, and Button Press Duration). Debounce, threshold, monitoring interval, and boot behavior are configured directly through YAML substitutions.
- **`default`** — Full functionality: controllable/dimmable Blue and Red LED entities with automation options, Early Publish, power-monitoring scaling entities, button configuration, debounce/threshold/reboot-behavior configuration entities, and diagnostics (Restart Firmware, IP Address, Uptime, Button Press Duration).

The old `plus` behavior is now the `default` profile, while the old base/default behavior is available as `lite`. The old `update` and `factory` variants used to build OTA-update and factory-flash binaries have not been ported into this package structure yet.

Lite can also be selected with a single package include by using the matching hardware convenience entrypoint:

```yaml
packages:
  Kauf.Plug: github://KaufHA/kauf-plugs/packages/kauf-plf12-lite.yaml
```

## Control Entities

***Kauf Plug*** switch entity — The main switch that controls the plug's relay. It will be named just Kauf Plug, or whatever you set the `friendly_name` substitution to in yaml. All other entities will be prefixed with the name of this entity plus whatever is indicated below for the particular other entity.

***Blue LED*** switch entity *(`lite` and `default`)* — Controls the blue LED on and off. In `minimal`, the blue LED exists but simply follows the relay state automatically with no controllable entity.

***Red LED*** switch entity *(`lite` and `default`)* — Controls the red LED on and off. In `minimal`, the red LED exists but only blinks as a Wi-Fi/Home Assistant connection-status indicator with no controllable entity.

## Configuration Entities

The LED, button, scaling, and Early Publish configuration entities are available in `lite` and `default`; entities marked `default` only are the additional Plus-style settings. `minimal` has no configuration entities beyond the plug's own relay switch. Entities listed as disabled by default can be enabled in Home Assistant, or simply modified through the web interface by clicking "Visit Device" in Home Assistant or typing the plug's IP address into a web browser. Adding the substitution `disable_entities: "false"` in your yaml file will cause all entities to be automatically enabled in Home Assistant.

***Blue LED Brightness*** number entity — Sets the brightness of the blue LED when on.

***Blue LED Config*** select entity — Configures the behavior of the blue LED. Has the following options. Defaults to *Power Status*.
- *Power Status*: LED follows relay state.
- *No Automation*: The plug will never automatically change the LED, but the LED can still be user-controlled via the switch entity.
- *Invert Power Status*: LED follows the inverse of the relay's state (LED on when relay is off).
- *Error Status*: The LED will blink when an error is detected. The most common errors are being unable to connect to Wi-Fi or Home Assistant.
- *Error and Power*: The LED will follow the relay state, but then also blink when an error is detected.
- *Error and Invert Power*: The LED will follow the inverse of the relay state, but then also blink when an error is detected.

***Red LED Brightness*** number entity — Sets the brightness of the red LED when on.

***Red LED Config*** select entity — Same as the Blue LED select entity, but for the Red LED. Defaults to *Error Status*.

***Button Config*** select entity — Defines when the button toggles the relay. To disable the button toggling the relay, select the *Don't Toggle* option. Otherwise, you can select the relay to toggle either on press or on release of the button. Toggle on release might be desired in order to have a different action performed when the button is held for a certain amount of time — for instance, toggling can be blocked in case the button is held for 30 seconds to reenable the Wi-Fi AP. Defaults to *Toggle on Press*.

***Early Publish*** switch entity *(`kauf` components with `lite` or `default`)* — Controls whether the plug will report power usage before the configured update interval if a change greater than configured thresholds is detected. Restores its last-saved state, defaulting **off** on first boot. Default thresholds are +/- 3W or 25%.

***Scale Current***, ***Scale Power***, ***Scale Voltage*** number entities — Disabled by default. Scale the respective sensor. Can be used to calibrate the plug.

***Debounce Time*** number entity *(`default` only)* — Defines an amount of time that the button needs to be held before toggling the relay. Minimum and default values are model-specific: PLF10 defaults to 100ms (50ms minimum); PLF12 defaults to 20ms (10ms minimum). Changes take effect without a reboot.

***Use Threshold*** number entity *(`default` only)* — Disabled by default. Sets a threshold (in watts) for the *Device In Use* binary sensor. The binary sensor turns on if the power detected by the plug exceeds the threshold.

***Monitoring Update Interval*** select entity *(`default` only)* — Disabled by default. Defines the update frequency for the power monitoring sensor entities. Defaults to *10s*. Hard-coded options are *2s*, *5s*, *10s*, *30s*, and *60s*. The *YAML Configured* option uses the value defined by the `sub_update_interval:` substitution in YAML, which defaults to 20s. Changing this value will cause the plug to automatically reboot to effect the change.

***Boot State*** select entity *(`default` only)* — Defines the relay's behavior on boot. Defaults to *Restore Power Off State*.
- *Restore Power Off State*: The relay will restore its state from the last time the plug was changed before power off or reboot.
- *Invert Power Off State*: The relay will toggle every time it is rebooted or powered off and back on.
- *Always On*: The relay will always turn on when the plug boots.
- *Always Off*: The relay will always remain off when the plug boots.
- *YAML Configured*: The boot state is set using the `sub_restore_mode:` substitution to any valid value for an [ESPHome switch's restore mode](https://esphome.io/components/switch/).

***No HASS*** switch entity *(`default` only)* — Disabled by default. Turn this switch on if not using Home Assistant with the plug. Prevents the plug from flashing an error due to no API connection and rebooting every 15 minutes.

## Sensor Entities

***Power***, ***Current***, ***Voltage*** sensor entities — Power-monitoring readings from the plug's power-monitoring chip.

***Total Daily Energy*** sensor entity — Reports total energy used so far each day, resets at midnight.

***Device In Use*** binary sensor entity *(`lite` and `default`)* — Indicates whether the plugged-in device is running. `default` uses the *Use Threshold* entity; `lite` uses the `sub_threshold` YAML substitution.

## Diagnostic Entities

Diagnostic entities are available in `lite` and `default` and are disabled by default. Adding the substitution `disable_entities: "false"` in your yaml file will cause all entities to be automatically enabled in Home Assistant.

***Restart Firmware*** button entity *(`lite` and `default`)* — Press this button to reboot the plug.

***IP Address*** sensor entity *(`lite` and `default`)* — Gives the plug's IP address.

***Uptime*** sensor entity *(`lite` and `default`)* — Gives the plug's uptime in seconds.

***Button Press Duration*** sensor entity *(`default` only)* — Indicates the amount of time in milliseconds that the button was last pressed for. Reads zero while the button is being pressed.

## Advanced Settings (set via YAML substitutions)

You can configure the following aspects by adding substitutions to your own yaml. Adding one of these will overwrite the corresponding default from the packages above.

***name*** — Defines the plug's name in the ESPHome dashboard, the device name in Home Assistant, mDNS URL, and other various aspects. The name should use only lower-case letters and dashes. Do not use spaces or underscores.

***friendly_name*** — The friendly name will be used to name every entity in Home Assistant. Add a substitution to change this to something descriptive for each device. Capital letters and spaces are good for the friendly name.

***component_family*** — Selects the component implementation. Defaults to `kauf`.
- `kauf` — Kauf custom components, persistence stamping, deprecation checks, and Kauf-only functionality.
- `stock` — Stock ESPHome components without Kauf customizations; some functionality, such as early-publish tuning, is unavailable.

***component_source*** — Selects where the `kauf` family loads [`KaufHA/common`](https://github.com/KaufHA/common) from. Defaults to `release` and is ignored by the `stock` family.
- `release` — The pinned release declared in `packages/shared/kauf-versioning.yaml`.
- `beta` — The latest commit on the repository's default branch, refreshed hourly.
- `local` — A sibling local checkout for component development. Expects `KaufHA/common` next to this repository.

***profile*** — Selects which profile to use. See [Profile Options](#profile-options).

***sub_restore_mode*** — Defines the state of the relay on boot-up. See more about the options under restore_mode here: [GPIO Switch](https://esphome.io/components/switch/gpio.html). In the `default` profile, the *Boot State* select entity must remain in the *YAML Configured* option for this substitution to be effective.

***disable_entities*** — Adding a substitution to redefine this to `"false"` will result in all entities being automatically enabled in Home Assistant.

***button actions*** — The `sub_on_press` and `sub_on_release` substitutions can be used to define scripts that will run based on button press and release if you want something different than the default actions to happen.

***plug state actions*** — The `sub_on_turn_on` and `sub_on_turn_off` substitutions can be used to define scripts that will run when the plug's relay turns on and off, respectively.

***sub_toggle_check*** — Defines a script that, if running, will stop the button from toggling on release. Used to stop a button-press action from occurring when the 30-second hold-to-reset action executed instead.

***sub_on_hold_30s*** — Executes when the button has been held for 30 seconds, while the button is still being held. Defaults to the script that force-enables the Wi-Fi AP (see [Clearing Wi-Fi Credentials](#clearing-wi-fi-credentials-getting-wi-fi-ap-to-reconfigure-credentials) below).

***sub_default_scale_power***, ***sub_default_scale_current***, ***sub_default_scale_voltage*** — Define default values for the power, current, and voltage scaling number entities.

***power monitoring calibration*** — The values used to calibrate power monitoring can be overwritten using substitutions to make the calibration more accurate: `current_resistor_val`, `voltage_divider_val`, `sub_hlw_model`, and each of `power_cal_val1_in`/`power_cal_val1_out`/`power_cal_val2_in`/`power_cal_val2_out` (and the equivalent `current_cal_*`/`voltage_cal_*` pairs), which define two calibration points per measurement.

***sub_update_interval*** — Defines the time delay between updates of the power monitoring sensors. In `default`, the *Monitoring Update Interval* select entity needs to be set to *YAML Configured* for this substitution to take effect; in `lite` and `minimal`, it directly sets the interval.

***sub_threshold*** — Defines the power threshold in watts for the *Device In Use* binary sensor in `lite`. In `default`, the *Use Threshold* entity controls this value instead.

***sub_pm_initial_option*** — Defines the default option for the power monitoring update interval select entity. `default` profile only.

***sub_early_publish_percent*** — Defines a percentage change, relative to the current power output, at which the plug will ignore the configured update interval and immediately report a new power output. Defaults to 25%.

***sub_early_publish_percent_min_power*** — Defines a minimum power, under which the early-publish-percent configuration will be ignored. Defaults to 0.5W.

***sub_early_publish_absolute*** — Defines an absolute power change after which the plug will ignore the configured update interval and update immediately. The unit for this is **not watts** but rather a raw value used under the hood for processing. To figure out what value you need here, you can enable verbose logging and the HLW component will output the needed raw value. It's model-specific and already set to approximate a 3W change by default: 16.68 for PLF10 (1W ≈ 5.56), 4.05 for PLF12 (1W ≈ 1.35).

***sub_hlw_timeout*** — Sets an amount of time to wait for a signal from the power monitoring chip before assuming the power being used is 0W. Defaults to 3s.

***sub_debounce*** — Requires an integer without units. Defines the debounce time in milliseconds. It directly controls debounce in `lite` and `minimal`; in `default`, debounce is instead controlled by the *Debounce Time* config entity.

> **Deprecated:** `sub_reboot_timeout`, `default_button_config`, and `disable_webserver` are no longer functional substitutions in this package structure — setting them will trigger a deprecation warning at compile time pointing to [`KaufHA/common`'s deprecation notes](https://github.com/KaufHA/common/blob/main/DEPRECATED_SUBSTITUTIONS.md).

### Wi-Fi networks

Multiple Wi-Fi networks can be configured with the [`networks:`](https://esphome.io/components/wifi.html#connecting-to-multiple-networks) key under `wifi:`. By default, the configured `networks:` will be added in addition to the default `initial_ap` network. If you set your SSID/password without the `networks:` key, that automatically replaces the default `initial_ap` network.

With the `kauf` component family, the Kauf Wi-Fi component also supports `only_networks: true` under `wifi:`. This discards the packaged `initial_ap` network and uses only the entries supplied under `networks:`. This option is not available with the `stock` component family. `fast_connect` defaults to disabled and does not need to be explicitly disabled for multiple networks.

## Factory Reset (`kauf` family only)

With the `kauf` component family, going to the plug's URL in a web browser and adding `/reset` will completely wipe all settings from flash memory. Note that this will wipe the plug's memory of whether the relay was on or off, and therefore the plug may turn off. This custom endpoint is not available with the `stock` component family.

## Clearing Wi-Fi Credentials, Getting Wi-Fi AP to Reconfigure Credentials

In the `lite` and `default` profiles, holding the plug's button for 30 seconds will clear any programmed Wi-Fi credentials, including credentials hard-coded in YAML, and cause the plug to put its Wi-Fi AP back up. With the `kauf` component family, going to the plug's URL in a web browser and adding `/clear` will do the same thing. The `/clear` endpoint is not available with the `stock` component family. No other settings or data will be lost.

## Troubleshooting

### Binary Size Error

ESPHome added API encryption by default, which can make plug binary files too big to OTA update. If you get the message `ERROR Error binary size: Error: ESP does not have enough space to store OTA file.`, we recommend that you remove API encryption by commenting out or deleting the following lines:

```yaml
# api:
#   encryption:
#     key: ...
```

If you want to keep API encryption, you can flash first with the `minimal` profile each time you need to update or upgrade, and then revert back to your desired profile.

### Additional Troubleshooting

General troubleshooting ideas applicable to all products are located in the [Common repo's readme](https://github.com/KaufHA/common/blob/main/README.md#troubleshooting).

## Recommended Tasmota Template

***Important Note:*** Do not have anything plugged into the plug when first flashing Tasmota. The default Tasmota template uses the relay for an indicator LED, so the relay will turn on and off every second or so until you change the template or get the plug connected to Wi-Fi.

**PLF12:**
```
{"NAME":"Kauf Plug","GPIO":[0,320,0,32,2720,2656,0,0,321,224,2624,0,0,0],"FLAG":0,"BASE":18}
```

**PLF10:**
```
{"NAME":"KAUF Plug","GPIO":[576,0,320,0,224,2720,0,0,2624,32,2656,0,0,0],"FLAG":0,"BASE":18, "CMND":"SwitchDebounce 100", "CMND":"ButtonDebounce 100"}
```

This page explains how to use the template if you need help: https://templates.blakadder.com/howto.html

## Notes

- Targets select model-specific pins/calibration/product metadata via `<model>-common.yaml`. See [Include Tree](#include-tree) for the full resolution path, including how the power backend is selected.
- PLF10 and PLF12 source repos were not modified.

## Links

- [Purchase PLF12 plugs on Amazon](https://www.amazon.com/dp/B0BJLGNPPX)
- [Purchase PLF10 plugs on Amazon](https://www.amazon.com/dp/B09JQ3LRHB)
- [KAUF YouTube Channel](https://www.youtube.com/channel/UCjgziIA-lXmcqcMIm8HDnYg)
- [KaufHA Website (PLF12)](https://kaufha.com/plf12)
- [KaufHA Website (PLF10)](https://kaufha.com/plf10)
