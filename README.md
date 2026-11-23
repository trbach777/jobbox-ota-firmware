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

ESP1 0.1.33+34 reads one APM10 on C1 (direct I2C, SDA GPIO4 / SCL GPIO5)
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

Current ESP2 is 0.8.25+36. ESP1 0.1.23+24 and later support updates entirely
from the HMI Updates page. Older ESP1 firmware needs the original maintenance
procedure or a one-time USB bootstrap. Keep power on until reboot and validation finish.

AUTO configuration is shared by the HMI (Control > AUTO settings), paired web
dashboard and Android app 0.1.4. Set thresholds, transition delays and each fan's
PWM at the three stages, then save. ESP1 stores the settings in NVS and restores
them after restart. AUTO start/pause from paired clients does not require HMI confirmation.

- HMI: Control, choose PWM, Apply, Confirm. The C1 AUTO button starts or pauses automatic testing.
- B1 GPIO2: increase fan 2 (common extraction).
- B2 GPIO8: mute the buzzer for five minutes only. Another click restarts the
  five-minute interval; it never turns the buzzer on. Timed voice buzzer commands
  remain available and B2 cancels an active timed signal.
- B3 GPIO21: increase fan 3 (local extraction).
- B4 GPIO39: toggle C1 AUTO on/off. Turning AUTO off stops all three fans.
- B5 GPIO41: increase fan 1 (supply).
- A fan key selects local MANUAL control and preserves the other two fan duties.
  Press B4 to return to AUTO. The GPIO wiring is unchanged.
- Supply fan 1 has a 75-percent minimum: OFF remains 0%, requests 1-74% become
  75%, and 75-100% are unchanged. The floor covers AUTO (including saved settings),
  remote manual commands and physical buttons, and is reported in actual output.
  Saved AUTO settings are not rewritten. Stopped/fault/OTA/maintenance outputs
  remain stopped. B5 cycles through 0/75/80/100/0 percent.
- Fan 2 cycles through 0/20/40/60/80/100/0 percent. Fan 3 respects the current
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

## Persistent dust reports and safety handbook (2026-11-21)

Update ESP2 to 0.8.22+33 first, then ESP1 to 0.1.31+32. Both require the
normal HMI update confirmation. No device installation has been verified for
this release; the owner has powered the model off until the next day.

ESP1 keeps the existing per-boot raw CSV log and adds checksummed daily `.jds`
checkpoints on SD. Each sensor contributes only a newly observed valid sample.
Dates and Monday-to-Sunday weeks use Vietnam time (UTC+7). Reports contain
sample mean, minimum, maximum, first peak time, sample count, and minutes with
observations. Missing/offline data is not zero. These are observed-sample
statistics, not full-shift occupational exposure measurements.

The assistant and paired web/Android API can query a rolling 35-day window.
Current-day data is provisional. Allow about 70 seconds after startup to sync
all historical dates from ESP1. Checkpoints normally flush each minute; an
abrupt power loss can lose recent uncommitted statistics. CRC recovery ignores
torn/corrupt trailing checkpoints but cannot guarantee against SD filesystem
or hardware damage. No old CSV import, automatic log deletion, or cloud upload
is added. New statistics start after this firmware is installed with a usable
SD and a valid RTC/network clock. Existing CSV files are preserved.

Rabit gains `self.jobbox.get_dust_history` and `self.jobbox.get_safety_guide`.
The bundled Vietnamese handbook covers textile dust, sewing ergonomics,
machine maintenance, electrical safety, fire/evacuation, and interpretation
of JOB BOX readings. It uses reviewed HSE, OSHA and GOV.UK guidance. It is
reference material alongside the workshop's own risk assessment and training;
JOB BOX is not a fire alarm or occupational exposure compliance instrument.
The text can be read locally; cloud assistant Q&A still needs Internet.

Android APK 0.1.6 is prepared separately and is not distributed in this
firmware-only repository. Older Android APKs do not show the new reports and
handbook sections.

## Daily private Google Drive backup and SD retention (2026-11-23)

Current pair: ESP2 0.8.23+34 (install first), ESP1 0.1.32+33. Cloud backup
is DISABLED by default and cannot operate until the owner deploys the prepared
Google Apps Script receiver and provisions its HTTPS /exec URL and random key
through the paired, trusted HTTPS JOB BOX web page. No private cloud URL/key,
Google credentials, flash backup, or NVS is included in this distribution.
No real Google deployment, board installation or upload is verified yet.

ESP1 rotates raw logs into date/boot/part CSV files below 64 KiB, then backs up
finished days from 00:10 Vietnam time (UTC+7). If powered off or offline it
catches up when available. The network task is separate from the sensor loop;
only the existing logger task touches SD, with one bounded backup job in flight.
The uploader verifies TLS and exactly binds each server ACK to device, file,
byte count and SHA-256. ContentService response redirects are followed as GET
only to the Google content host; the POST/key is never replayed there.

A retention sweep is due every seven calendar days, checkpointed in NVS.
Raw logs must be at least seven days old and JDS summaries at least 35 days
old (the owner's confirmed choice). Before deletion, the receiver must freshly
confirm the remote bytes again and ESP1 must recheck the source. Prior SD ACK
receipts alone cannot authorize deletion. Failed uploads, absent SD, low memory,
changed destinations or invalid clock keep local data. No forced card format.
Retention can exceed the minimum because sweeps are periodic and incomplete
backup prevents deletion. A prolonged backup failure can still fill the card.

The receiver places data under a private owner Drive folder, then device/date
subfolders. It produces a daily file index and readable Vietnamese summary JSON.
The receiver/setup instructions are prepared in the local source project at
cloud/google-drive; they are not deployed automatically by a firmware update.
Existing run_*.csv logs from earlier versions remain untouched and are not
imported automatically. Rabit's local history query window remains 35 days;
this release does not add querying long-term Drive history from the assistant.

Web gains Drive provisioning and status. APK 0.1.7 adds status only (configuration
requires trusted HTTPS web), and is distributed separately from this repository.
Optional ESP1 N16R8 octal PSRAM is initialized at 40 MHz; a missing external
memory is tolerated, and allocation/upload failures keep the SD source.
The existing control pins, C1 AUTO, button map, 75% supply minimum and OTA startup
sensor validation policy are retained. No physical retention test is claimed.

## Live Google Sheets (2026-11-23)

ESP1 seals raw CSV segments each minute and prioritizes the latest segment for upload.
The private Google receiver can upsert one last-observation row per minute, grouped
by day/month. It needs the new LiveSheets code, Sheets permission and deployment.
Missing samples stay blank; retries do not duplicate rows. Web and APK 0.1.8 expose
the validated private Sheet link after a successful acknowledgement.
No endpoint, backup key, Sheet ID or sensor data is included in these public images.

## HTTPS transport fix (ESP2 0.8.25+36)

Static assets, JSON and audio use bounded 1024-byte HTTP chunks to avoid
intermittent TLS responses truncated at the first 4096-byte record. Static
assets close their connection after transfer to release the limited HTTPS slots.
ESP1 remains 0.1.33+34. Backup account configuration is still required.
