# JOB BOX OTA firmware

Firmware images for the two ESP32-S3 N16R8 boards in JOB BOX.

- ESP1: sensors and fan controller. Updates require local maintenance mode.
- ESP2: TJC HMI and Xiaozhi voice assistant.
- Each board checks its own manifest. Installation requires confirmation on the HMI.
- Image identity, size, build sequence and SHA-256 are checked before activation.
- Initial installation of an OTA-capable partition table requires USB.

This distribution contains application images and manifests only. Wi-Fi credentials
are provisioned on the devices. Flash backups and NVS data are not distributed.
See release_audit.json for the exact versions and SHA-256 values.

## Current ESP1 commissioning profile

The current ESP1 release reads only the installed APM10 on C1. C2/C3 BGSY210 sensors are not installed, and fan outputs remain unarmed. The voice buzzer uses GPIO15 independently, with a bounded on-time and an ESP1 acknowledgement. ESP2 retains HMI button conversation and a 1.5-second silence endpoint.
