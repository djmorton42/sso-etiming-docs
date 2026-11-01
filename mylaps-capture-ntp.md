---
title: "MYLAPS clock synchronization through Capture"
description: "Configure and maintain local DNS and NTP on Capture for a MYLAPS decoder on an isolated timing LAN."
---

# MYLAPS clock synchronization through Capture

## Purpose and current status

Our MYLAPS ProChip decoder needs an accurate clock while operating on a timing LAN without internet access. The Windows 11 computer **Capture** has both a connection to that LAN and a separate connection with internet access.

Capture obtains time from an internet time server and serves its own clock to the decoder using **NTP (Network Time Protocol)**. Because the decoder does not expose a configurable NTP server address, we use **Technitium DNS Server** on Capture to redirect its existing NTP hostname to Capture.

This configuration has been tested successfully with the MYLAPS decoder. Packet captures confirmed DNS redirection and bidirectional NTP communication, and the complete setup was subsequently confirmed working. The exact clock offset has not been measured.

This page documents decoder-to-Capture synchronization. It does not configure or establish synchronization with a FinishLynx camera, the start gun, or any other timing device. Those clocks and start-signal paths must be checked separately.

## Network configuration

| Component or setting | Value |
| --- | --- |
| Timing computer | Capture, running Windows 11 |
| Capture's timing-LAN address | `192.168.0.25` |
| Possible decoder addresses | `192.168.0.19` or `192.168.0.20` |
| Decoder's configured DNS server | `192.168.0.25` |
| Decoder's observed NTP hostname | `us.pool.ntp.org` |
| Local DNS override | `us.pool.ntp.org` → `192.168.0.25` |
| Capture's upstream time source | `time.windows.com,0x9` |
| DNS software on Capture | Technitium DNS Server |
| NTP software on Capture | Built-in Windows Time service (`W32Time`) |
| Technitium administration console | <http://localhost:5380/> on Capture |

Devices use manually assigned IP addresses. Internet Connection Sharing, DHCP, and a route from the decoder to the internet are not required. Capture and the decoder must be able to communicate on the timing LAN; a Wi-Fi access point must not isolate them from each other.

## How it works

1. Capture synchronizes its clock with `time.windows.com` through its internet connection.
2. The decoder asks Capture's DNS server for the address of `us.pool.ntp.org`.
3. Technitium answers with `192.168.0.25` instead of a public NTP server address.
4. The decoder sends NTP requests directly to Capture on UDP port 123.
5. Windows Time replies using Capture's clock, and the decoder uses those replies to synchronize.

DNS only selects the server address. It is not involved in the subsequent NTP timestamp exchange and adds no ongoing timestamp offset.

**Only the decoder uses Capture as its DNS server.** When Technitium offered to configure Capture's network adapters to use itself, we selected **No**. Capture retains its existing DNS settings for internet access. Editing Capture's Windows hosts file alone would not supply DNS answers to the decoder.

## Setup or rebuild

Perform clock changes and synchronization outside an active race. Commands below run on Capture in **PowerShell as Administrator**.

### 1. Confirm network connectivity

Confirm Capture still has `192.168.0.25` on its timing-LAN adapter and that its separate internet connection works:

```powershell
ipconfig /all
```

On the decoder, set the DNS server to `192.168.0.25`. Leave a secondary DNS server blank if possible, so the decoder cannot bypass the override. Keep its normal IP address, subnet mask, and other timing-network settings.

### 2. Configure Technitium

Install [Technitium DNS Server](https://technitium.com/dns/) on Capture if needed, then open <http://localhost:5380/>. Complete the administrator account setup and change any default password.

Create the override:

1. Open **Zones** and choose **Add Zone**.
2. Set the zone name to `us.pool.ntp.org` and the type to **Primary Zone**.
3. Leave other options at their defaults and create the zone.
4. Open that zone and choose **Add Record**.
5. Save the following record:

| Record field | Value |
| --- | --- |
| Type | `A` |
| Name | `@` |
| IPv4 address | `192.168.0.25` |
| TTL | `60` seconds |

`@` means the exact zone name, so this record answers queries for `us.pool.ntp.org`. Retain the automatically created SOA and NS records. If the zone or A record already exists, inspect or edit it rather than adding a duplicate.

Under **Settings → General → DNS Server Local End Points**, make sure DNS listens on `192.168.0.25:53`, or on the default all-interface IPv4 listener, `0.0.0.0:53`. Labels may differ slightly by Technitium version.

The decoder also requested `mylaps.com` and `api-gateway.mylaps.com`. Do not redirect those names; they are separate from this NTP configuration. Resolving their public addresses does not give the decoder internet access.

### 3. Enable the Windows NTP server

Windows normally obtains time as a client. Enable its separate server provider so it can answer the decoder:

```powershell
Set-ItemProperty `
  -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\W32Time\TimeProviders\NtpServer' `
  -Name Enabled -Value 1

Set-Service -Name W32Time -StartupType Automatic
Restart-Service -Name W32Time
```

Keep the Windows NTP client enabled. No change to `AnnounceFlags` was part of the documented setup.

### 4. Configure Windows Firewall

Allow DNS and NTP requests from either decoder address. This script creates the named rules if missing and updates them if they already exist:

```powershell
$captureIP = '192.168.0.25'
$decoderIPs = @('192.168.0.19', '192.168.0.20')

$rules = @(
  @{ Name = 'MYLAPS DNS UDP'; Protocol = 'UDP'; Port = 53 },
  @{ Name = 'MYLAPS DNS TCP'; Protocol = 'TCP'; Port = 53 },
  @{ Name = 'MYLAPS NTP';     Protocol = 'UDP'; Port = 123 }
)

foreach ($rule in $rules) {
  $parameters = @{
    DisplayName   = $rule.Name
    Direction     = 'Inbound'
    Action        = 'Allow'
    Enabled       = 'True'
    Protocol      = $rule.Protocol
    LocalPort     = $rule.Port
    LocalAddress  = $captureIP
    RemoteAddress = $decoderIPs
    Profile       = 'Any'
  }

  if (Get-NetFirewallRule -DisplayName $rule.Name -ErrorAction SilentlyContinue) {
    Set-NetFirewallRule @parameters
  } else {
    New-NetFirewallRule @parameters
  }
}
```

`Profile Any` lets these rules work even when Windows classifies the isolated LAN as Public. The rules are limited to Capture's LAN address and the two decoder addresses. Existing unrelated firewall rules remain in effect.

### 5. Verify Capture's upstream synchronization

```powershell
w32tm /query /source
w32tm /resync
w32tm /query /status
```

The source observed on Capture was `time.windows.com,0x9`. This is a valid setting: `0x9` combines NTP client mode with a fixed polling interval. It does not refer to DNS or the decoder's IP address.

Check for a successful resync and a recent **Last Successful Sync Time**. A source of **Local CMOS Clock** does not confirm internet synchronization.

### 6. Verify DNS and synchronize the decoder

```powershell
nslookup us.pool.ntp.org 192.168.0.25
```

The answer must give `192.168.0.25`. A `Server: Unknown` label is harmless if the returned address is correct; it usually means the DNS server has no reverse DNS name.

Restart the decoder if it is still using an old cached answer, then select **Sync using NTP** on the decoder. Confirm that the decoder reports successful synchronization.

## Verification with Wireshark

Capture on Capture's **timing-LAN adapter**, using this display filter:

```text
(ip.addr == 192.168.0.19 || ip.addr == 192.168.0.20) && (dns || udp.port == 123)
```

Expected traffic:

| Direction | Protocol | Expected content |
| --- | --- | --- |
| Decoder → Capture | DNS | Query for `us.pool.ntp.org` |
| Capture → Decoder | DNS | Answer containing `192.168.0.25` |
| Decoder → Capture | NTP | Client request to UDP port 123 |
| Capture → Decoder | NTP | Server response from UDP port 123 |

The successful capture showed **NTP version 3** client requests and server responses. That version is expected and interoperates with Windows Time.

Select an NTP server reply and expand **Network Time Protocol**. For ordinary synchronized service, expect a leap indicator other than `3` (unsynchronized) and a stratum between `1` and `15`. Stratum indicates distance from the reference clock, not a measured accuracy rating.

**A reply does not alone prove the decoder accepted the time.** Check the decoder's success indication too. A short request-to-response gap captured on Capture measures local turnaround, not full network round-trip time or clock agreement.

## Meet-day checks

- Confirm Capture has `192.168.0.25` and the decoder has one of the documented addresses.
- Confirm the Technitium service and Windows Time service are running after startup.
- Keep Capture awake while providing DNS and NTP.
- Check Capture's time source and last successful synchronization.
- Trigger and confirm decoder synchronization before racing.
- Verify the separate start-signal and photofinish synchronization procedures.

DNS, registry, and firewall settings persist across restarts. Verify both services actually start after a reboot before depending on unattended operation.

## Troubleshooting

| Symptom | Check or action |
| --- | --- |
| DNS query times out | Check Technitium is running, its port 53 listener includes the LAN address, and firewall rules allow the decoder's current IP. |
| DNS returns a public address | Check the exact zone name and root A record, the decoder's configured DNS server, secondary DNS settings, and cached answers. |
| DNS is correct but no NTP request appears | Trigger synchronization, check decoder status, and restart it if necessary outside racing. |
| NTP requests arrive but no replies leave | Check Windows Time is running, the server provider is enabled, and UDP 123 is allowed. |
| NTP replies arrive but synchronization fails | Inspect leap indicator, stratum, and timestamps in the reply; check Capture's upstream synchronization and the decoder's error message. |
| Works for `.20` but not `.19` | Confirm firewall rules include both addresses. |
| Works until reboot | Check service startup and that Capture's LAN IP has not changed. |
| Works over Ethernet but not Wi-Fi | Check access-point client isolation, LAN bridging, signal quality, and packet loss. |

### Windows reports “no time data was available”

This means Capture could not obtain usable upstream time; it does not by itself identify a failure in decoder DNS or local NTP serving. During setup, a later retry of `w32tm /resync` succeeded without changing the existing `0x9` setting.

Inspect and test first:

```powershell
w32tm /query /source
w32tm /query /peers
Resolve-DnsName time.windows.com
w32tm /stripchart /computer:time.windows.com /dataonly /samples:5
```

Offsets from `stripchart` demonstrate NTP replies are reachable. However, stripchart uses an ephemeral UDP source port while the normal Windows Time client uses source port 123, so one can work while the other encounters firewall restrictions.

Retry discovery and synchronization:

```powershell
w32tm /resync /rediscover
```

If source configuration needs repair on a standalone computer, the following is an optional troubleshooting change, not a required part of the working configuration:

```powershell
w32tm /config /manualpeerlist:"time.windows.com,0x8" /syncfromflags:manual /update
Restart-Service W32Time
w32tm /resync /rediscover
```

On a domain-managed computer, check the organization's time policy before replacing its source. Do not configure Capture to obtain time from its own `us.pool.ntp.org` override.

## Accuracy and timing implications

Successful NTP synchronization establishes a shared time reference; it does not prove an exact error bound. Millisecond-level agreement can be possible on a quiet wired LAN, but our decoder-to-Capture offset has not been measured. Wi-Fi contention, retransmissions, and unequal delays in the two directions can increase error substantially.

Use Ethernet where practical when tight agreement matters. Whether this decoder automatically repeats synchronization or only synchronizes on command has not been established here. Do not infer continuous synchronization from repeated packets alone; those could also be retries or manually triggered attempts.

NTP conveys UTC. Display time zones and daylight-saving settings are separate. A whole-hour display difference should prompt a check of those settings before changing NTP.

Capture and the decoder may closely agree while both differ from UTC. Conversely, a decoder synchronized once can drift relative to Capture afterward. If Capture loses internet access, the local network path can remain available, but clock drift and Windows' advertised synchronization state need checking; continued acceptable service is not guaranteed.

The decoder timestamps detections locally. Synchronizing it with Capture does not remove network delivery delays, synchronize a separate camera clock, or substitute for a shared start impulse.

## Changing the configuration

| Change | Settings to update |
| --- | --- |
| Capture's LAN IP changes | Technitium A record, decoder DNS setting, DNS listener if explicitly bound, firewall local addresses, and verification commands. |
| Decoder's IP changes | Firewall remote addresses, Wireshark filter, and any timing application's decoder connection settings. |
| Decoder firmware requests a different NTP hostname | Confirm with a DNS capture and add an equivalent override for the observed name. |
| Capture's upstream time source changes | Windows Time client configuration; decoder still points to Capture. |

To remove this setup, restore the decoder's intended DNS configuration, remove the `us.pool.ntp.org` zone, and remove the three named MYLAPS firewall rules. If Capture no longer needs to serve NTP, set its `NtpServer` provider's `Enabled` value to `0` and restart Windows Time. Preserve the Windows NTP client if Capture still needs internet time.

## References

- [Technitium DNS Server](https://technitium.com/dns/)
- [Technitium help topics](https://technitium.com/dns/help.html)
- [Microsoft: Windows Time service tools and settings](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings)
- [Microsoft: Troubleshoot “no time data was available”](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/error-message-run-w32tm-resync-no-time-data-available)
- [Wireshark: DNS](https://wiki.wireshark.org/DNS)
- [NTP clock filter algorithm](https://www.ntp.org/documentation/4.2.8-series/filter/)
