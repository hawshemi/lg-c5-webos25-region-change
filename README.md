# LG C5 / webOS 25 Region Change for 5 GHz Wi-Fi

> [!IMPORTANT]
> This is a **region configuration change**, not a firmware update or firmware fix. It will not repair a Wi-Fi hardware or router problem.

Use this guide when 5 GHz Wi-Fi is unavailable because of the TV's factory region configuration.

> [!WARNING]
> Changing the region can affect tuner behavior, the LG Content Store, installed or available apps, country settings, and warranty or service handling. A wrong area option may leave the TV with unsuitable regional settings.
>
> Read and save the original area option before writing anything. Continue at your own risk.

This procedure is for **Windows 11 and PowerShell**. It uses LG Developer Mode and the upstream [`lg-geolock-bypass`](https://github.com/lennylxx/lg-geolock-bypass) script. Root access is not required.

## Confirmed result

| Item | Confirmed value |
| --- | --- |
| TV | `LG OLED55C56LA.AMQQLJD` |
| Software | webOS 25 / `10.3.0-1902` |
| Firmware | `33.31.68` |
| Original configuration | Middle East, area option `4956` |
| Target configuration | EU, area option `3122`, country set to Germany |
| Result | 5 GHz Wi-Fi restored; channel 36 tested; all available 5 GHz channels worked |
| Other functions | No problems observed with the tuner, apps, or country settings |
| Rollback | Not tested |

The TV remained stable with Germany selected as its country. This result applies only to the configuration above. Other models, firmware versions, and hardware groups may behave differently.

## Before you start

### Requirements

- LG C5 TV running webOS 25
- TV and Windows 11 PC on the same local network
- LG Developer account
- PowerShell with `curl.exe`, `ssh.exe`, and `scp.exe`

This repository intentionally contains only this README. Download `change_region.sh` from its upstream project when instructed; do not add a copy to this repository.

## Procedure

Developer Mode → Key Server → SSH key → key permissions → script upload → save original value → write target value → reboot → verify → roll back if needed

### 1. Enable LG Developer Mode

Follow the official [LG Developer Mode documentation](https://webostv.developer.lge.com/develop/getting-started/developer-mode-app):

1. Install and open **Developer Mode** on the TV.
2. Sign in with your LG Developer account.
3. Turn **Dev Mode Status** on.
4. Let the TV reboot.
5. Open the Developer Mode app again.

Developer Mode has a limited session time. Extend it in the app if needed.

### 2. Enable Key Server

In the Developer Mode app, turn **Key Server** on. Note the case-sensitive six-character passphrase shown by the app.

Note the TV's IP address shown in the Developer Mode app.

In PowerShell, set the TV address and create a temporary working folder:

```powershell
$TV = "<TV_IP>"
$WorkDir = Join-Path $env:TEMP "lg-c5-region-change"
New-Item -ItemType Directory -Path $WorkDir -Force | Out-Null
Set-Location $WorkDir
```

Replace `<TV_IP>` before continuing.

Optional port checks:

```powershell
Test-NetConnection $TV -Port 9991
Test-NetConnection $TV -Port 9922
```

Both should report `TcpTestSucceeded : True`. if not, turn the TV on and off, then in the Developer app in TV turn the Key store option off and on (or on and off). Then try again.

### 3. Download `webos_rsa`

While Key Server is on, download the TV's SSH private key:

```powershell
curl.exe -f "http://${TV}:9991/webos_rsa" -o .\webos_rsa
Get-Item .\webos_rsa
```

### 4. Fix SSH key permissions if needed

Try the next step first. If OpenSSH reports that the private key permissions are too open, run:

```powershell
$CurrentUser = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
icacls.exe .\webos_rsa /inheritance:r
icacls.exe .\webos_rsa /grant:r "${CurrentUser}:(R)"
```

If `icacls.exe .\webos_rsa` still lists another user or group with access, remove that exact entry:

```powershell
icacls.exe .\webos_rsa /remove "DOMAIN\UserOrGroup"
```

### 5. Copy `change_region.sh`

Download the current script directly from the upstream project into the temporary folder:

```powershell
curl.exe -fL "https://raw.githubusercontent.com/lennylxx/lg-geolock-bypass/main/change_region.sh" -o .\change_region.sh
```

Copy it to the TV:

```powershell
scp.exe -i .\webos_rsa -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa -P 9922 .\change_region.sh "prisoner@${TV}:/tmp/change_region.sh"
```

Enter the six-character Developer Mode passphrase when prompted. On the first connection, confirm the TV's SSH host-key prompt only if the address is correct.

### 6. Read and save the original area option

Do this before changing the region:

```powershell
ssh.exe -T -i .\webos_rsa -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa -p 9922 "prisoner@$TV" "sh /tmp/change_region.sh read" | Tee-Object -FilePath .\original-area-option.txt
```

The first line contains the value to preserve:

```text
Current area option: <ORIGINAL_AREA_OPTION>
```

Save `<ORIGINAL_AREA_OPTION>` somewhere safe. Do not guess it later.

For the confirmed TV above, the original value was `4956`. Do not assume that value is correct for another TV.

### 7. Write the target area option

This guide uses area option `3122`. It is a **known EU value** (`hwSettingGroup=EU`) and worked on the confirmed configuration above. It is not universal and may be wrong for another TV.

The upstream project states that non-US config and settings mappings are best-effort. Review its current notes before proceeding.

Write `3122`:

```powershell
ssh.exe -T -i .\webos_rsa -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa -p 9922 "prisoner@$TV" "sh /tmp/change_region.sh 3122"
```

Check the output for a successful NVRAM write and verification. Stop if it reports a failure.

### 8. Wait for the automatic reboot

After the region is changed successfully, the TV will reboot automatically. Wait for it to turn on with the new region. Do not disconnect its power during the reboot.

### 9. Verify 5 GHz Wi-Fi

After the TV turns on, open **Settings > Network > Wi-Fi Connection** and check that 5 GHz networks are visible.

#### Roll back (if needed)

> [!CAUTION]
> Rollback has not been tested on the confirmed configuration. The command below follows the upstream method, but success is not confirmed for this TV.

Turn Key Server on and copy `change_region.sh` again if the TV has rebooted. Then restore the exact value saved in step 6.

The TV should reboot automatically after the original value is written. After reboot, restore the appropriate country in the TV settings and verify the original region.

## Troubleshooting

### Port 9991 is unavailable

Open the Developer Mode app, confirm Dev Mode Status is on, toggle **Key Server** off and on, and retry the download immediately. Confirm that `$TV` is correct and both devices are on the same network.

### Port 9922 is unavailable

Confirm Developer Mode and Key Server are on. Use port `9922`, not port `22`. Check `$TV`, local firewall or VPN rules, and router client isolation.

### Private key permissions are too open

Run the `icacls.exe` commands in step 4. The key should be readable by your Windows account, not by broad groups or other users.

### `Permission denied`

Use user `prisoner`, port `9922`, the downloaded `webos_rsa` file, and the current case-sensitive six-character passphrase. If needed, toggle Key Server and download `webos_rsa` again.

### `PTY allocation request failed`

The Developer Mode account may not provide an interactive terminal. Use `ssh.exe -T` with the remote command at the end, exactly as shown above.

### `/tmp/change_region.sh` disappeared after reboot

This is expected because `/tmp` is temporary. Repeat the `scp.exe` command from step 5 after each reboot.

### EZ-Adjust value reverts

Some webOS builds enforce the region through `factorymanager`, so an EZ-Adjust change can revert. Do not keep changing it in EZ-Adjust. Use the direct NVRAM method above and check the command output for a successful write.

## References

- [LG: App Testing with Developer Mode App](https://webostv.developer.lge.com/develop/getting-started/developer-mode-app)
- [`lennylxx/lg-geolock-bypass`](https://github.com/lennylxx/lg-geolock-bypass)
