# Mobile Image Upload Failure Caused by an MTU/PMTU Black Hole

Technical documentation for the Suntek HC-940Ultra, HC-950Ultra, and
HC-960Ultra camera family.

[English](MTU_PMTU_BLACK_HOLE_EN.md) |
[简体中文](MTU_PMTU_BLACK_HOLE_ZH-CN.md)

Document status: 2026-07-29

## 1. Summary

The HC-950Ultra can establish a mobile data connection with a Simbase IoT SIM
while image uploads remain permanently at `0%`. Cloud configuration and P2P
live video continue to work. A controlled A/B test showed that changing only
the CPU2 Linux mobile-interface MTU from 1500 to 1200 makes the same image
upload succeed immediately.

The behavior is consistent with a Path MTU Discovery (PMTUD) black hole:

1. The camera sends TCP segments suitable for an interface MTU of 1500.
2. The effective end-to-end MTU in the IoT, roaming, tunnel, or Internet
   breakout path is lower.
3. Oversized packets cannot cross the path.
4. The required ICMP feedback is filtered, lost, or not handled correctly.
5. TCP retransmits data that still cannot pass.
6. The HTTPS image upload stalls until an internal timeout, while the display
   remains at `0%`.

A persistent, event-driven MTU of 1300 is the recommended field workaround.
It has been confirmed on the HC-950Ultra after a cold restart and in both
tested camera operating modes. MTU 1200 remains the most conservative known
working value.

## 2. Scope and validation status

| Model and firmware date | Technical compatibility | Real-camera validation |
|---|---|---|
| HC-950Ultra, 2026-05-27 | Confirmed | MTU 1300 persists across restart and works in both tested operating modes |
| HC-940Ultra, 2025-04-23 | Confirmed by firmware inspection and installer simulation | Pending |
| HC-960Ultra, 2026-03-26 | Confirmed by firmware inspection and installer simulation | Pending |

All three inspected firmware versions contain the same four relevant network
scripts and the same legacy and active RIL network paths. A model- and
firmware-specific patch package is still required for each version because
the main network application differs between releases.

Do not treat static compatibility as real-device approval. The HC-940Ultra and
HC-960Ultra still require cold-boot, reconnect, DHCP-renewal, and upload tests
on physical cameras.

## 3. Confirmed symptoms

The issue was reproduced with the following network configuration:

- IoT provider: Simbase
- IMSI prefix: `20404`
- APN: `simbase`
- APN username and password: empty
- Roaming: enabled
- Mobile interface: normally `eth0`
- Default interface MTU: 1500

Observed behavior:

- Mobile data registration succeeds.
- Cloud configuration retrieval succeeds.
- The cloud control channel succeeds.
- P2P live video succeeds at approximately 90 kbit/s.
- JPEG upload over HTTPS remains at `0%`.
- A regular consumer SIM succeeds with MTU 1500 in the same camera.
- The MTU-1500 failure was observed while the IoT SIM used both Telekom and
  Vodafone as visited roaming networks.

The successful MTU-1350 test was performed while the IoT SIM was registered
on the Telekom network. MTU 1350 has not been independently validated on the
Vodafone roaming path because the camera cannot manually select a network.

## 4. Controlled A/B results

| Test condition | Result |
|---|---|
| Simbase IoT SIM, interface MTU 1500 | Image upload remains at `0%` |
| Same camera, SIM, session, APN, and cloud account; MTU changed to 1200 | Image upload succeeds |
| Simbase IoT SIM, MTU 1350, Telekom roaming path | Image upload succeeds |
| Persistent MTU 1300 on HC-950Ultra | Survives restart and works in both tested camera modes |
| Regular consumer SIM, MTU 1500 | Image upload succeeds within seconds |
| Cloud configuration with Simbase | Works |
| P2P live video with Simbase | Works |

Only the interface MTU was changed between the failed MTU-1500 upload and the
successful MTU-1200 upload. The SIM, APN, authentication, radio connection,
cloud account, and upload operation remained unchanged.

## 5. Network state and diagnostic commands

The original CPU2 Linux state showed an active `eth0` interface with an address
in `192.168.255.0/24`, a default gateway at `192.168.255.1`, and MTU 1500.

Check the current interface MTU:

```sh
cat /sys/class/net/eth0/mtu
```

Check whether the active DHCP client uses the RIL path:

```sh
ps w | grep '[u]dhcpc'
```

Check the kernel PMTUD mode:

```sh
cat /proc/sys/net/ipv4/ip_no_pmtu_disc
```

The controlled tests used:

```text
ip_no_pmtu_disc = 0
```

This is the normal PMTUD-enabled mode. The fact that a lower interface MTU was
still required supports the diagnosis that useful PMTU feedback does not reach
or does not correct the affected TCP flow.

Temporary diagnostic workaround:

```sh
ifconfig eth0 mtu 1200
cat /sys/class/net/eth0/mtu
```

This manual command is not persistent. An interface recreation, modem
reconnect, DHCP event, USB-network reset, or camera restart may return the MTU
to 1500.

## 6. Why only image upload fails

Firmware inspection indicates that different camera functions use different
communication paths:

- Cloud configuration and control exchange relatively small messages.
- P2P live video uses a separate P2P/UDP-oriented path and packetization.
- Image upload requests a signed upload URL and then sends the JPEG using a
  sustained HTTPS `PUT` transfer to object storage.

The HTTPS upload is much more likely to produce full-sized TCP segments and to
encounter a PMTU black hole. Successful control messages and P2P video
therefore do not prove that the TCP/TLS upload path can carry packets derived
from an MTU of 1500.

Reducing an application upload buffer is not an equivalent fix. TCP segment
size is derived from the interface MTU and negotiated MSS. The correction must
act on the interface MTU, TCP MSS, a modem-provided MTU, or TCP black-hole
recovery.

## 7. Persistent patch design

The patch uses a small configuration file:

```text
/etc/mtu_patch.conf
```

Example:

```text
MTU_VALUE=1300
```

Only tested values are accepted:

| MTU | Use |
|---:|---|
| 1200 | Most conservative confirmed workaround |
| 1300 | Recommended persistent value with safety margin |
| 1350 | Confirmed on the tested Telekom path, but with less margin |

The patch must cover both the legacy GSM path and the active RIL path:

```text
/usr/share/gsmscripts/vk_rndis_ecm.sh
/usr/share/gsmscripts/vk_udhcpc_script.sh
/usr/share/rilscripts/vk_rndis_ecm.sh
/usr/share/rilscripts/vk_udhcpc_script.sh
```

### 7.1 Interface-start hooks

The two ECM/RNDIS scripts must:

1. Read and validate the configured MTU.
2. Bring up only the expected mobile interface (`eth0` or `usb0`).
3. Apply the MTU immediately after the interface is brought up.
4. Read the resulting value from sysfs and fail if it differs.
5. Start DHCP only after the MTU is correct.

Expected diagnostic output:

```text
[VK NET] eth0 MTU=1300
```

### 7.2 DHCP-event hooks

The two DHCP scripts must reapply and verify the MTU on:

- `deconfig`
- `bound`
- `renew`

For `bound` and `renew`, the MTU must be set before CPU1 is notified that the
mobile network is ready. This prevents a cloud or image-upload connection from
starting during a short MTU-1500 window.

### 7.3 No watchdog

No periodic daemon or watchdog is required. Event-driven hooks cover interface
creation and DHCP changes without adding continuous wakeups to a
battery-powered camera.

## 8. Generic patcher requirements

The firmware-specific changes should be deployed by a generic,
model-specific patcher. Its transport and execution mechanism is outside the
scope of this document.

The patcher should:

1. Require the necessary system privileges.
2. Identify the exact camera model and firmware revision.
3. Verify the main network application against a known model-specific
   fingerprint.
4. Accept each target script only if it exactly matches either the known
   original or the known patched version.
5. Reject unknown or locally modified scripts.
6. Validate shell syntax before installation.
7. Create internal and removable-media backups of all four original scripts.
8. Write the MTU configuration atomically.
9. Stage each replacement in its target directory and replace it atomically.
10. Verify installed bytes and permissions.
11. Apply the selected MTU immediately if the mobile interface already exists.
12. Produce explicit success, failure, and diagnostic logs.
13. Support `install`, `verify`, and `restore` modes.
14. Recover safely from a mixed original/patched state after interrupted
    installation.

Each supported model and firmware version must have its own allow-listed
fingerprint and recovery data. Similar scripts alone are not sufficient reason
to patch an unknown release.

## 9. Why the first patch revision failed

The first implementation changed only:

```text
/usr/share/gsmscripts/
```

The installer completed and could change an already active `eth0` to MTU 1300,
but the normal camera connection later used the newer `[VK RIL]` state machine:

```text
/usr/share/rilscripts/
```

That active path recreated or configured the interface at MTU 1500. A success
result from the installer therefore did not prove persistence.

The corrected design patches all four scripts. The active RIL scripts enforce
the MTU before DHCP and again during DHCP events; the GSM scripts remain
patched as a legacy/fallback path.

## 10. Installation verification

After applying a model-specific patch:

1. Remove the patch media.
2. Insert the normal camera storage card.
3. Perform a complete cold start.
4. Wait for the mobile data connection.
5. Check:

```sh
cat /etc/mtu_patch.conf
cat /sys/class/net/eth0/mtu
ps w | grep '[u]dhcpc'
```

Expected values for the recommended configuration:

```text
MTU_VALUE=1300
1300
```

Confirm that the DHCP command refers to the active RIL DHCP script.

Then validate:

- at least 20 consecutive full-resolution image uploads;
- at least 20 uploads after another cold start;
- reduced-resolution uploads;
- a modem or mobile-network reconnect;
- a DHCP renewal;
- both camera operating modes;
- cloud configuration retrieval;
- P2P live video;
- a regular consumer SIM as a regression test.

After each restart or reconnect, verify the interface MTU again.

## 11. Restore behavior

Restore mode should:

1. Verify that every current target script is a known original or known patch.
2. Atomically restore all four original scripts.
3. Remove the persistent MTU configuration.
4. Restore an already active mobile interface to MTU 1500.
5. Verify the restored bytes and active value.
6. Preserve an external backup for manual recovery.

A later full vendor firmware update may also replace the CPU2 root filesystem
and remove the patch. The MTU must be checked again after any firmware update.

## 12. Recommended vendor implementation

An official firmware correction should apply the MTU in the common network
code instead of relying on a field patch. It should:

- read a network-provided MTU from the modem or PDP context when available;
- support a provider-specific or configurable APN MTU;
- use a safe fallback such as 1300 or 1200;
- reapply it after every interface recreation, reconnect, and DHCP event;
- apply it before cloud connections start;
- verify and log the active interface and MTU;
- optionally enable TCP MTU probing if supported by the kernel;
- retry timed-out HTTPS uploads with a lower effective MTU/MSS;
- preserve queued images and retry them after recovery;
- report DNS, TCP, TLS, HTTP, and upload-stage errors separately.

## 13. Limitations and privacy

- Runtime validation is complete only for the listed HC-950Ultra firmware.
- HC-940Ultra and HC-960Ultra require physical-camera validation.
- MTU 1350 is confirmed only on the tested Telekom roaming path.
- A future firmware release may use different scripts or event ordering.
- Camera, UART, and modem logs may expose IMEI, IMSI, ICCID, MAC addresses,
  access tokens, and signed upload URLs. Redact them before publication.
