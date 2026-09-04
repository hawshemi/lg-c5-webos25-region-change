# LG C5 / webOS 25: Enable 5 GHz Wi-Fi by Changing the TV Region

This guide is for LG C5-class TVs where **5 GHz Wi-Fi is unavailable because of the factory region configuration**.

It uses LG Developer Mode and the open-source `lg-geolock-bypass` script to change the area option directly in NVRAM. **Root/Homebrew is not required.**

> [!WARNING]
> Changing the TV region can affect tuner standards, LG Content Store availability, apps, language/country choices, and other region-specific features.
>
> **Always read and save the original area option before changing anything.**
>
> Do not use random area codes. The `3122` example below is the known **EU hardware-group** area option and is not automatically correct for every LG model or regional variant.

## Requirements

- LG C5-class TV running webOS 25
- TV and Windows PC on the same local network
- [LG Developer Mode app](https://webostv.developer.lge.com/develop/getting-started/developer-mode-app)
- Windows 11 with PowerShell, `curl`, `ssh`, and `scp`
- [`lg-geolock-bypass`](https://github.com/lennylxx/lg-geolock-bypass)
  - Required file: `change_region.sh`

The upstream project was tested by its author on an **LG OLED C5, webOS 10.3.0 / firmware 33.30.97**.

## 1. Download the script

Download or clone:

https://github.com/lennylxx/lg-geolock-bypass

Extract it, then open **PowerShell in that folder**.

You should see at least:

```text
change_region.sh
calc_area.py
README.md
```

## 2. Enable Developer Mode on the TV

On the TV:

1. Install and open **Developer Mode**.
2. Sign in.
3. Turn **Dev Mode Status** ON.
4. Allow the TV to reboot if requested.
5. Open Developer Mode again.
6. Turn **Key Server** ON.
7. Note:
   - TV IP address
   - 6-character passphrase

Keep the Developer Mode app open while getting the key.

In the commands below, replace:

```text
<TV_IP>
```

with the TV's IP address.

Optional connection check:

```powershell
Test-NetConnection <TV_IP> -Port 9991
Test-NetConnection <TV_IP> -Port 9922
```

Both should show:

```text
TcpTestSucceeded : True
```

## 3. Download the TV SSH key

Run:

```powershell
curl.exe http://<TV_IP>:9991/webos_rsa -o .\lg_private.key
```

Confirm the key exists:

```powershell
Get-Item .\lg_private.key
```

If port `9991` suddenly stops responding, turn **Key Server OFF and ON again** on the TV and retry the `curl` command immediately.

## 4. Fix private-key permissions if Windows rejects it

If `ssh` or `scp` reports:

```text
WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions ... are too open.
```

run:

```powershell
$me = "$env:USERDOMAIN\$env:USERNAME"
icacls .\lg_private.key /inheritance:r
icacls .\lg_private.key /grant:r "$($me):(R)"
```

If the error names another Windows user/group that still has access, remove that exact entry:

```powershell
icacls .\lg_private.key /remove "DOMAIN\UserOrGroup"
```

## 5. Copy the script to the TV

Run:

```powershell
scp -i .\lg_private.key `
  -o HostKeyAlgorithms=+ssh-rsa `
  -o PubkeyAcceptedAlgorithms=+ssh-rsa `
  -P 9922 `
  .\change_region.sh prisoner@<TV_IP>:/tmp/
```

When asked for a passphrase, enter the **6-character passphrase shown in the TV Developer Mode app**.

## 6. Read and save the original area option

Do this **before writing anything**:

```powershell
ssh -T -i .\lg_private.key `
  -o HostKeyAlgorithms=+ssh-rsa `
  -o PubkeyAcceptedAlgorithms=+ssh-rsa `
  -p 9922 `
  prisoner@<TV_IP> "sh /tmp/change_region.sh read"
```

Example output:

```text
Current area option: 4956
  continentIdx:     92
  languageCountry:  6
  hwSettingGroup:   1
```

Your values may be different.

**Save the complete output.** You need the original area option if you ever want to restore the TV.

## 7. Change the region

For the EU hardware group, the known area option is:

```text
3122
```

It decodes as:

```text
continentIdx:    50
languageCountry: EU
hwSettingGroup:  EU
```

To write it:

```powershell
ssh -T -i .\lg_private.key `
  -o HostKeyAlgorithms=+ssh-rsa `
  -o PubkeyAcceptedAlgorithms=+ssh-rsa `
  -p 9922 `
  prisoner@<TV_IP> "sh /tmp/change_region.sh 3122"
```

A successful result should contain something similar to:

```text
[+] NVRAM contiArea2All set to 3122
[+] Verify: ... "contiArea2All":"3122" ...
[+] Decoded: ... lang=EU hw=EU ...
```

The TV may reboot automatically after the change.

## 8. After reboot

Go to:

```text
Settings → General → System → Location
```

If needed, select the appropriate country in the new region.

Then open:

```text
Settings → Network → Wi-Fi Connection
```

5 GHz networks should now be visible if the original problem was the regional Wi-Fi restriction.

Connect normally and confirm the TV shows the connection as **5G / 5 GHz**.

## Optional: verify the new area option

If Developer Mode is still active, copy `change_region.sh` again if the TV has rebooted because `/tmp` is cleared on reboot.

Then run:

```powershell
ssh -T -i .\lg_private.key `
  -o HostKeyAlgorithms=+ssh-rsa `
  -o PubkeyAcceptedAlgorithms=+ssh-rsa `
  -p 9922 `
  prisoner@<TV_IP> "sh /tmp/change_region.sh read"
```

It should report the new area option.

You can also verify it in the LG service menu if you already have legitimate service-menu access.

## Roll back to the original region

Use the **exact original area option saved in step 6**.

Example:

```powershell
ssh -T -i .\lg_private.key `
  -o HostKeyAlgorithms=+ssh-rsa `
  -o PubkeyAcceptedAlgorithms=+ssh-rsa `
  -p 9922 `
  prisoner@<TV_IP> "sh /tmp/change_region.sh <ORIGINAL_AREA_OPTION>"
```

Do not guess the original value.

## Troubleshooting

### `curl: (7) Failed to connect ... port 9991`

On the TV:

1. Open Developer Mode.
2. Confirm **Dev Mode Status** is ON.
3. Turn **Key Server OFF**, then ON.
4. Retry the `curl` command immediately.

### `Permission denied (publickey,keyboard-interactive)`

Usually one of these:

- `lg_private.key` was not downloaded
- the key permissions are too open
- the wrong key is being used
- Key Server / Developer Mode was restarted and the key needs to be downloaded again

### `PTY allocation request failed on channel 0`

Do not open an interactive shell.

Use `ssh -T` with the command at the end, as shown in this guide.

### The area option changes in EZ-Adjust but reverts when leaving

Some newer webOS builds enforce a region lock through `factorymanager`.

That is the reason this method writes the value directly through the low-level storage service instead of relying on EZ-Adjust.

### Script disappeared after reboot

Normal. `/tmp` is temporary.

Copy `change_region.sh` to the TV again before running it.

## References

- LG Developer Mode documentation:  
  https://webostv.developer.lge.com/develop/getting-started/developer-mode-app

- `lg-geolock-bypass` project and area-code table:  
  https://github.com/lennylxx/lg-geolock-bypass
