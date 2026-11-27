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

Current ESP2 is 0.8.27+38. ESP1 0.1.23+24 and later support updates entirely
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

## Today's average assistant query (ESP2 0.8.26+37)

Adds a dedicated current-day statistics tool using the device date in UTC+7.
General history defaults to a day when period is absent, while explicit weekly
and dated reports are preserved. A recognized today's-average question redirects
a mistaken live-reading tool call to the recorded daily sample sum/count.
Spoken results include each location, missing samples, partial-day and stale-cache
notes. Host regression and firmware build passed; live voice acceptance remains pending.
ESP1 remains 0.1.33+34; its retained statistics and backup configuration are used.

## Daily statistics diagnostics (ESP2 0.8.27+38)

Recognizes composed/decomposed and uppercase Vietnamese today-average questions.
Daily reports distinguish a pending ESP1 summary, no samples, partial samples and
a future date. At midnight, a pending new-day packet is not labelled a future day.
Authenticated state adds compact ai_tools diagnostics (tool name, turn and history
status/date) to separate a missing tool call from unavailable recorded statistics.
No configuration, statistics or secrets are reset. Live acceptance is still pending.

## C1 startup offset trial (ESP1 0.1.34+35 / ESP2 0.8.28+39)

Display-only trial: after 60 seconds of warm-up and with all fan commands off,
collect 16 fresh C1/C2 pairs over at least 30 seconds. Reject an unstable difference
(standard deviation above 10 ug/m3), then freeze delta = mean(C1 raw - C2 raw).
C1 live display becomes max(0, C1 raw filtered - delta). It does not track C2 later.
Place both intakes together in the same direction before learning.

HMI C1 detail can toggle the trial; web and Android 0.1.9 can enable/relearn or
turn it off. Only the enabled flag persists, while delta is relearned on reboot.
The trial defaults enabled in this release. Live values are explicitly labelled.
AUTO, SD logs, daily/weekly statistics, history charts and Google Sheets/Drive
remain raw; no stored measurements are rewritten. Live assistant reports explain
this distinction. This relative adjustment is not a traceable calibration.

Host regressions, firmware builds and rendered HMI layouts passed. Hardware
learning and installation have not yet been confirmed. Update ESP2 first, then
ESP1 using confirmation on the HMI; the current ESP1 profile stages maintenance
automatically. This distribution contains firmware and manifests only.

## Immediate C1 offset and quiet HMI (ESP1 0.1.35+36 / ESP2 0.8.30+41)

Supersedes the 16-pair startup learning window: ESP1 latches delta = C1 raw - C2 raw
as soon as each channel has a fresh, checked measurement. It does not wait for
fan outputs to stop or for an additional averaging/warm-up gate. Missing samples,
failed reads, samples older than 3 seconds and future timestamps cannot seed delta.
The normal sensor display/control warm-up remains in place. Delta is frozen until
reboot or explicit relearn; later C2 changes do not drag C1 along.

The HMI now shows the adjusted live number with normal sample age/details, without
compensation status, delta or a compensation button. Web/API and Android 0.1.10
retain raw/adjusted diagnostic values and enable/relearn/disable controls. ESP2
accepts both the new single-pair READY state and the previous 16-pair protocol.
Install ESP2 first, then ESP1 using HMI OTA confirmation. Existing NVS settings,
including a previously disabled compensation flag, are preserved.

Raw AUTO inputs, SD logs, Google Drive/Sheets, charts and daily/weekly statistics
are unchanged. No historical records are rewritten. This is relative alignment;
first-frame sensor transients can affect the result, and it is not traceable
calibration. Place C1/C2 intakes together before learning.

Both firmware builds, host regressions, HMI rendering and mobile/desktop diagnostic
panel tests passed. Hardware installation and physical learning remain unverified.
The current 'Xưởng sản xuất may mặc' HMI/web title is preserved. Public files contain
only deployable firmware, manifests, release checks and documentation.

## HMI waits for all three sensors (ESP2 0.8.31+42)

The overview and sensor detail pages only show numerical PM readings when C1,
C2 and C3 are all valid and fresh in the current ESP1 state. Until then all
sensor readings show --, with a count of available channels and channel fault
hints. A healthy channel does not display its number while another channel is
missing. After any channel becomes invalid or stale, all readings are hidden
again; the complete group returns automatically when all three recover.

The gate requires an ESP1 snapshot younger than 3 seconds, valid nodes, and
sensor ages plus snapshot elapsed time strictly below 15 seconds. It does not
change sensor polling or make the physical sampling times identical. Existing
normal sensor warm-up and C1 display-only offset remain in place. Raw control,
logging, statistics, cloud sync and web diagnostics are unchanged. Fan and buzzer
controls remain available according to their existing rules.

ESP1 remains 0.1.35+36 with the same image hash. Update only ESP2 through HMI
OTA confirmation. Host tests passed 2,623 assertions; ESP32-S3 build passed;
22 rendered HMI pages had no layout issues. Hardware installation is pending.

## Direct assistant reports (ESP2 0.8.32+43)

Removes the unconditional paragraph about startup display compensation from live
dust tool results. History tool output keeps its raw-measurement basis and compact
alignment metadata, rather than injecting a spoken trial notice. Tool descriptions
and daily answer instructions request direct answers and no unsolicited parameter
adjustment announcements; explanations of adjustment remain appropriate when the
user specifically asks about it or a discrepancy between live and recorded values.

This release changes assistant context only. It does not hide real missing-sensor
or stale-data errors, alter readings, change C1 offsets, or alter AUTO/control/logs.
The HMI waits for all three valid sensors exactly as in 0.8.31+42. ESP1 remains
0.1.35+36. The same assistant context is used by the box, web and Android clients.
No server account or cloud voice setting is changed.

Host/UI/audio/speech and daily/weekly statistics regressions passed. Hardware
installation and the final model-generated spoken response still need verification.
Update ESP2 using the existing HMI OTA confirmation.

## Restore unadjusted C1 display (ESP1 0.1.36+37)

The previous first-frame C1-C2 startup offset could exceed a later filtered C1
reading and clip its display to zero. The sensors are installed at different
locations, so their difference does not establish a calibration offset.

ESP1 now defaults that trial OFF and ignores the legacy automatic enabled flag.
Only a new explicit manual_enabled flag can enable the existing trial again.
Missing/read-error/invalid flags keep the trial off. On first upgrade C1 returns
to its original filtered value; no artificial minimum or equalization is added.
Actual measured zero remains zero. Later explicit requests still use the existing
authenticated boot-bound command and ESP1 acknowledgement.

Sensor acquisition, raw values, filters, AUTO, other settings, SD/cloud/statistics
are unchanged. ESP2 remains 0.8.32+43 with its existing three-sensor display gate.
This does not calibrate any sensor or assert which physical sensor is accurate.
Host regression and ESP32-S3 build results are recorded in the release audit;
physical raw/display comparison and hardware installation remain pending.
Update ESP1 through the HMI OTA confirmation.

## Per-location dust regulation reference (ESP2 0.8.33+44)

Adds compare_dust_standard(sensor1..3) for fresh raw PM2.5 with sensor location,
source links and an explicit INSUFFICIENT_FOR_COMPLIANCE conclusion. QCVN05:2023/
BTNMT references 24-hour PM2.5 at45 ug/Nm3 from2026 (50 before2026), and excludes
indoor air (1.1.3). Instant unstandardized model readings are not 24-hour legal
measurements. QCVN02:2019/BYT workplace cotton dust TWA1mg/m3 cannot be replaced
by optical PM2.5; mixed cotton/polyester requires correct material/method review.
No safe/unsafe certification is generated. The numeric relation is only a reference.
Unknown A/B/C mapping asks for the actual location; stale/missing samples return
no invented value. Raw, corrected live values, AUTO thresholds and records remain
unchanged. Shared regulation knowledge is available in the web and Android0.1.11.

Official source links and review date2026-11-26 are embedded. Updates are not
automatically researched by the device. Host boundary/stale/zero/location tests,
existing controller UI/history tests and ESP32-S3 build passed. Installation,
server tool selection and actual spoken answers remain unverified. Touch-position
fault remains a separate pending investigation; this release does not change it.
Update ESP2 through HMI OTA confirmation. ESP1 stays0.1.36+37.

## Comparison first and promotional phrase filter (ESP2 0.8.34+45)

For fresh readings at any of the three named locations, the dust-reference tool
now returns COMPARISON_READY, signed difference, percentage and ratio, with a
suggested answer leading with higher/lower/equal. Scope is explained briefly at
the end. Missing/stale samples still require a new sample. Raw PM2.5, location,
year-specific reference and no-compliance-verdict constraints are preserved.

A bounded Vietnamese case/accent-normalized filter rejects known channel-
subscription boilerplate in cloud STT before chat publication and in TTS
sentence_start before reply handling. Rejected turns close their audio session,
discard queued data and unpublished fan proposals, and resume conversation.
Genuine commands and user questions/complaints about the phrase remain allowed.
The root cause of the reported phrase has not been diagnosed from live logs;
this finite filter cannot identify all unwanted content or recall audio already
played before a server sentence notification arrives. Phone client transcripts
may also be shown locally before cloud STT reaches the device.

Host UI, audio, phrase-filter, numerical-reference and history checks passed;
ESP32-S3 build passed. Hardware installation, cloud tool selection and physical
audio acceptance remain pending. Update ESP2 via HMI OTA; ESP1 stays 0.1.36+37.

## All three workshop locations and explicit comparison basis (ESP2 0.8.35+46)

The comparison tool accepts sensor0 or an omitted sensor for all three locations
in one call. Reports every fresh raw value with its installed location; ranks
highest/lowest including ties and reports the range. Missing channels are named,
never replaced with zero, and do not suppress healthy channels.

MCP comparison basis now distinguishes workplace_regulation (default for circular
and workshop safety questions), workshop_thresholds (actual acknowledged ESP1
AUTO t1/t2), and explicit ambient_reference requests. It no longer silently
answers workplace questions using outdoor standards. Workplace output includes
QCVN02:2019/BYT under TT02/2019/TT-BYT and retains the distinction between optical
PM2.5 and a cotton-dust shift TWA measurement. It does not calculate a fabricated
legal pass/fail. Internal operating warning levels are labeled as configured
thresholds, never called statutory exposure limits. Sensor/fan/controller logic
and recorded raw data are unchanged.

Host group/partial/tie/zero/threshold/location/freshness tests, UI/audio/promotion
regressions and history tests passed. Catalog JSON and message-size checks passed.
ESP32-S3 build passed; Android0.1.13 shares the updated knowledge. Runtime server
tool selection and actual audio acceptance remain pending after OTA installation.
Update ESP2 via HMI confirmation. ESP1 remains0.1.36+37.

## HMI link recovery and AUTO stop (ESP2 0.8.36+47)

Stop AUTO now permits a fresh ESP1 link even if the HMI heartbeat is offline.
Boot/epoch, output arming, pending-command, maintenance and update checks remain;
start AUTO and other control paths retain their HMI requirements. Cloud set_auto
accepts the bare alias and its catalog explicitly maps stop to on=false. Success
still requires an ESP1 ACK. Web and Android0.1.14 allow the stop button in this
case, while retaining normal requirements to start AUTO.

HMI drawing/UART is managed by a bounded single-owner session. After missing ACK
or reset, it restores bkcmd, watchdog heartbeat and coordinate reporting, verifies
a numeric probe, and redraws. A scene-completion barrier rejects touches against
controls not yet drawn; subsequent live-data refreshes cannot prolong that barrier.
Authenticated diagnostics report UART/probe/touch/timeout/recovery counters,
coordinates and AUTO call/acceptance flags. No coordinate rotation, calibration,
HMI TFT update, sensor/fan logic or ESP1 image is changed.

Validation:3314 UI assertions, all rendered pages through the actual bounded
queue(maximum70/128 commands), transport fault/recovery and sustained-traffic
tests, audio and comparison regressions passed. Headless browser fixtures for web
and Android bridge verified on=false/current identity when HMI is unavailable.
ESP32-S3 build and signed APK checks passed. This is a software-path repair and
adds diagnostics; no live USB trace proves the cause of the user's touch failure.
Installation, physical touches and actual AI/ESP1 stop acceptance remain pending.
Update ESP2 by HMI OTA when responsive. ESP1 stays0.1.36+37.

## Responsive fixed navigation tabs (ESP2 0.8.37+48)

Bottom navigation now acts on the initial press. Release is consumed and cannot
choose an adjacent tab. Missing release no longer blocks navigation; same-tab
press noise is debounced for160ms, and reselecting the active page does not
restart full drawing. The fixed strip can navigate during scene drawing; body
controls still wait for the frame ACK and normal press/release validation. Body
hits use the original press coordinates after validation. Modal pages and pending
AUTO-save guards remain. No coordinate calibration or rotation is changed.

Live0.8.36 diagnostics showed healthy UART/no timeouts during the recorded tab
trial, but one press/release pair rejected during drawing. A user-labeled left
Tab tap was received inside its expected bounds, while paired press/release
counts were temporarily unequal and page change waited for release. This supports
a software sensitivity/timing repair; it is not a full touch-panel diagnosis.

Validation:3342 UI assertions, including all five tabs, missing/shifted release,
duplicate press, active-page no redraw, modal/body-control isolation, and all
rendered pages via the bounded transport(maximum70/128commands). Existing HMI
recovery, audio, promotion and dust-comparison checks passed. ESP32-S3 build passed.
Installation and physical tab acceptance remain pending. Update ESP2 by HMI OTA.
ESP1 remains0.1.36+37 and Android remains0.1.14.
