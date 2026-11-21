# JOB BOX OTA firmware

Firmware images for the two ESP32-S3 N16R8 boards in JOB BOX.

- ESP1: sensors and fan controller. HMI-initiated updates stop the outputs and enter maintenance automatically.
- ESP2: TJC HMI and Xiaozhi voice assistant.
- Each board checks its own manifest. Installation requires confirmation on the HMI.
- Image identity, size, build sequence and SHA-256 are checked before activation.
- Initial installation of an OTA-capable partition table requires USB.

This distribution contains application images and manifests only. Wi-Fi credentials
are provisioned on the devices. Flash backups and NVS data are not distributed.
See release_audit.json for the exact versions and SHA-256 values.

## Current three-sensor profile

ESP1 0.1.28+29 reads one APM10 on C1 (direct I2C, SDA GPIO4 / SCL GPIO5)
and two BGSY210 sensors on C2 (UART2 RX GPIO9) and C3 (UART0 RX GPIO38).
Each BGSY210 TXD must pass through a 3.3 V logic buffer. The ESP1 console uses
native USB Serial/JTAG so UART0 is reserved for C3. Sensor warm-up takes about
60 seconds; OTA startup validation requires valid readings from all three sensors
within 180 seconds.

All fans start off. Use the HMI Control page to start or pause C1 AUTO.
C2/C3 are included in displays, charts, assistant reports and SD logging.
AUTO remains based on filtered C1 PM2.5 and stops the fans on invalid C1 data.
Previously saved AUTO settings are restored from NVS. Fresh-device defaults
are 15 and 35 µg/m³; these are model-test settings, not occupational exposure limits.

Current ESP2 is 0.8.21+32. ESP1 0.1.23+24 and later support updates entirely
from the HMI Updates page. Older ESP1 firmware needs the original maintenance
procedure or a one-time USB bootstrap. Keep power on until reboot and validation finish.

AUTO configuration is shared by the HMI (Control > AUTO settings), paired web
dashboard and Android app 0.1.4. Set thresholds, transition delays and each fan's
PWM at the three stages, then save. ESP1 stores the settings in NVS and restores
them after restart. AUTO start/pause from paired clients does not require HMI confirmation.

- HMI: Control, choose PWM, Apply, Confirm. The C1 AUTO button starts or pauses automatic testing.
- B1 GPIO2: toggle C1 AUTO on/off. Turning AUTO off stops all three fans.
- B2 GPIO8: mute the buzzer for five minutes only. Another click restarts the
  five-minute interval; it never turns the buzzer on. Timed voice buzzer commands
  remain available and B2 cancels an active timed signal.
- B3 GPIO21 / B4 GPIO39 / B5 GPIO41: increase fans 1 / 2 / 3 respectively.
  A fan key selects local MANUAL control and preserves the other two fan duties.
  Press B1 to return to AUTO.
- Fans 1/2 cycle through 0/20/40/60/80/100/0 percent. Fan 3 respects the current
  30-percent minimum and cycles through 0/30/40/60/80/100/0 percent.
- Fan keys are blocked during maintenance, OTA or an I/O fault; B2 still mutes.
- Buttons use internal pull-ups: connect each normally open switch from its GPIO
  to common GND. External 10k pull-ups remain useful with long/noisy wiring.
- Each button is debounced independently for 80ms; hold about 0.3s and release.
  One event is generated per qualified release, with no key repeat. A boot-held
  key or read error/sample gap over 350ms requires stable release before rearming.
- B1+B5 held for three seconds retains the maintenance shortcut; that chord
  suppresses individual release actions. Software cannot reject every long EMI pulse.
- HMI manual leases last three seconds; loss of renewals switches fans off.
- Local button changes invalidate older HMI manual commands.
- Local button control remains independent of Wi-Fi and HMI connectivity.

Voice buzzer control and HMI button conversation with a 1.5-second silence endpoint remain available.
