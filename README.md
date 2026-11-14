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

## Current C1 test profile

ESP1 0.1.24+25 reads the installed APM10 on C1. All fans start off.
Use the HMI Control page to start or pause C1 AUTO. The test driver uses
filtered PM2.5 thresholds of 15 and 35 µg/m³. Fans stop on invalid C1 data.
These values are for model testing. Workplace exposure requires occupational sampling.

Current ESP2 is 0.8.21+32. ESP1 0.1.23+24 and later support updates entirely
from the HMI Updates page. Older ESP1 firmware needs the original maintenance
procedure or a one-time USB bootstrap. Keep power on until reboot and validation finish.

AUTO configuration is shared by the HMI (Control > AUTO settings), paired web
dashboard and Android app 0.1.4. Set thresholds, transition delays and each fan's
PWM at the three stages, then save. ESP1 stores the settings in NVS and restores
them after restart. AUTO start/pause from paired clients does not require HMI confirmation.

- HMI: Control, choose PWM, Apply, Confirm. The C1 AUTO button starts or pauses automatic testing.
- B1 GPIO2: AUTO/MANUAL. B2 GPIO8: fan 1. B3 GPIO21: fan 2. B4 GPIO39: fan 3.
- B2-B4 increase local duty and wrap back to zero. B5 GPIO41 mutes the buzzer.
- HMI manual leases last three seconds; loss of renewals switches fans off.
- Local button changes invalidate older HMI manual commands.
- Local button control remains independent of Wi-Fi and HMI connectivity.

Voice buzzer control and HMI button conversation with a 1.5-second silence endpoint remain available.
