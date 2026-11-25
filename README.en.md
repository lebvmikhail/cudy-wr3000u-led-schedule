# Cudy WR3000U — scheduled LED control on OpenWrt

A practical guide to configuring the seven Cudy WR3000U indicators in LuCI and switching them off overnight, then restoring their normal triggers in the morning.

**Default schedule:** LEDs off at 21:30 and restored at 07:00, using the router's local time.

> **Tested configuration:** Cudy WR3000U (256 MB), OpenWrt 25.12.5, build string `r33051`, LuCI branch `26.180.75667`. The author tested this method on this configuration. Operation on other models, hardware revisions, or firmware builds is not guaranteed.

- [Русская версия](README.md)
- [Verification and rollback checklist](docs/verification-and-rollback.md)
- [Documentation license — CC BY 4.0](LICENSE)
- [Original 4PDA post](https://4pda.to/forum/index.php?showtopic=1123233&view=findpost&p=145076946)

## What this setup does

1. Recreates a seven-entry LED map in LuCI so the morning restore reapplies both steady indicators and network activity triggers.
2. Turns off all LED class devices that expose a `brightness` attribute from 21:30 until 07:00.
3. Reapplies the LED configuration at 07:00.
4. Repeats the night-time shutdown after a router reboot: if power is lost overnight, the next cron run turns the indicators off again.

The cron loop targets every `/sys/class/leds/*/brightness` entry, not just the seven indicators listed below. This is intentional in the author's tested setup.

## Before you start

- Create a configuration backup in LuCI (**System → Backup / Flash Firmware**) and keep a copy of the current LED configuration if you know how to access it over SSH.
- Make sure the router's date, time, and time zone are correct. Cron uses the router's local system time.
- LED names and network interface names (`blue:wan`, `phy0-ap0`, `phy1-ap0`, etc.) depend on the device, driver, and firmware build. Do not apply this table to another router without checking its own LED and interface names.
- Writing `0` to a Linux LED `brightness` attribute clears the active trigger. The morning task must therefore reapply the LED configuration, rather than merely setting brightness to a non-zero value. See the [Linux LED sysfs ABI](https://docs.kernel.org/ABI/testing/sysfs-class-led).
- The additional `/etc/init.d/system restart` command is retained from the author's tested configuration. It reapplies base settings from OpenWrt's system configuration; it is not the LED-specific restoration mechanism. `/etc/init.d/led restart` performs the LED configuration restore.

## 1. Recreate the LED map in LuCI

Open **System → LEDs** (LED Configuration).

1. Remove the existing LED configuration rows.
2. Click **Save & Apply** to apply the deletion.
3. Add the seven rules in the table below. Keep the row order; it follows the author's physical left-to-right indicator order.
4. After adding all rows, click the blue **Save & Apply** button at the bottom of the page.

### LED rules

| Order | Rule name | LED (`sysfs`) | Trigger | Device | Modes |
|---:|---|---|---|---|---|
| 1 | `01_Power` | `blue:power` | `default-on` | — | — |
| 2 | `02_Internet_Blue` | `blue:wan-online` | `none` | — | — |
| 3 | `02_Internet_Red` | `red:fault` | `default-on` | — | — |
| 4 | `03_WAN` | `blue:wan` | `netdev` | `wan` | TX, RX |
| 5 | `04_LAN` | `blue:lan` | `netdev` | `br-lan` | Link, TX, RX |
| 6 | `05_WiFi_2.4G` | `blue:wlan-2ghz` | `netdev` | `phy0-ap0` | Link, TX, RX |
| 7 | `06_WiFi_5G` | `blue:wlan-5ghz` | `netdev` | `phy1-ap0` | Link, TX, RX |

### Rule notes

- `default-on` keeps the indicator continuously on.
- `netdev` links an LED to the state and activity of the specified network device. In Linux the modes are `link`, `tx`, and `rx`.
- `none` is selected for `blue:wan-online`. This is retained from the author's tested configuration. The observation that additional behavior for this indicator may be handled by Cudy firmware is specific to this build; it is not a general property of the OpenWrt `none` trigger.
- The author observed a faint red glow around the empty WAN window and attributes it to optical bleed from the adjacent `red:fault` LED. This observation is specific to the tested enclosure and unit; `blue:wan` is configured separately in software.

## 2. Configure the cron schedule

Open **System → Scheduled Tasks** and add the following block.

```cron
# Minute-by-minute night-time safeguard in case the router reboots after a power loss
* * * * * hour=$(date +\%H); minute=$(date +\%M); if [ "$hour" -gt 21 ] || { [ "$hour" -eq 21 ] && [ "$minute" -ge 30 ]; } || [ "$hour" -lt 7 ]; then for led in /sys/class/leds/*; do [ -f "$led/brightness" ] && echo 0 > "$led/brightness"; done; fi

# Main LED shutdown at 21:30
30 21 * * * for led in /sys/class/leds/*; do [ -f "$led/brightness" ] && echo 0 > "$led/brightness"; done

# Restore the LED configuration at 07:00
0 7 * * * /etc/init.d/led restart; /etc/init.d/system restart
```

### How the schedule works

- The **minute-by-minute safeguard** is active from 21:30 inclusive until 07:00 exclusive. After an overnight reboot, the next cron run turns the LEDs off again.
- The **21:30 task** performs a separate main shutdown. It duplicates the safeguard at the start of the night period and is retained from the author's tested configuration.
- The **07:00 task** runs `/etc/init.d/led restart` to reapply the LED configuration from `/etc/config/system`, including activity triggers. The additional `/etc/init.d/system restart` is retained from the author's tested combination; it is not the primary LED restoration command.

> **Important — escaping `%`:** in a cron command, `%` must be escaped with a backslash (`\%`). Otherwise cron may treat it as a command/input separator. The `%H` and `%M` tokens in this configuration are already escaped.

After saving the schedule, restart the **cron** service in **System → Startup**. Make sure the service is enabled and running.

## 3. Changing the schedule

For example, to turn the LEDs off at 23:00 and restore them at 06:00, update all three time conditions consistently:

- in the minute-by-minute safeguard, use `([ "$hour" -ge 23 ] || [ "$hour" -lt 6 ])`;
- change the second entry to `0 23 * * *`;
- change the third entry to `0 6 * * *`.

Keep the `\%` escaping in the `date` command.

## 4. Expected behavior and verification

- At 21:30 the indicators turn off.
- They remain off during the night interval.
- After an overnight reboot, some LEDs may illuminate during startup; the next minute-level cron run should turn them off again.
- At 07:00 the LED configuration is reapplied, restoring steady indicators and network activity triggers.

See the [verification and rollback checklist](docs/verification-and-rollback.md) for the test sequence and recovery steps.

## Compatibility and limitations

- Tested by the author on **Cudy WR3000U (256 MB), OpenWrt 25.12.5, build string `r33051`, LuCI branch `26.180.75667`**.
- Not tested on other Cudy models, hardware revisions, or third-party OpenWrt builds.
- LED names and network interface names can differ between devices and firmware builds.
- The cron task writes to `/sys/class/leds`; it controls system LED devices and requires the privileges provided by the router's cron service.

## Authorship and acknowledgements

**Author and editor:** Mikhail Lebedev, with AI assistance (ChatGPT).

The technical approach builds on posts by **zerocash**, **skafin**, and moderator **a43** on the 4PDA forum. Thanks to them for the groundwork and ideas used in the resulting configuration.

**Original post:** [4PDA — post #145076946](https://4pda.to/forum/index.php?showtopic=1123233&view=findpost&p=145076946).

## Technical references

- [Linux LED class documentation](https://docs.kernel.org/leds/leds-class.html)
- [Linux sysfs ABI: LED brightness and trigger](https://docs.kernel.org/ABI/testing/sysfs-class-led)
- [OpenWrt 25.12.5: `/etc/init.d/led`](https://github.com/openwrt/openwrt/blob/v25.12.5/package/base-files/files/etc/init.d/led)
- [OpenWrt 25.12.5: `/etc/init.d/system`](https://github.com/openwrt/openwrt/blob/v25.12.5/package/base-files/files/etc/init.d/system)
- [OpenWrt: UCI system configuration](https://git.openwrt.org/project/web/tree/docs/uci_system.txt)

## License

The original documentation created for this repository — text, explanations, translation, tables, and editorial arrangement — is licensed under **Creative Commons Attribution 4.0 International (CC BY 4.0)**. See [LICENSE](LICENSE) for the scope and details.

This license does not relicense third-party material or grant rights the author does not hold. In particular, posts and prior contributions by **zerocash**, **skafin**, and **a43** on 4PDA, external documentation, photographs, trademarks, firmware, and software remain subject to their own rights and terms.
