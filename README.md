# Ping from Host Device

A Zabbix template that monitors the reachability of remote devices via ICMP ping, executed locally by Zabbix Agent 2 on a host with network access to those devices. Designed for scenarios where the Zabbix server or proxy cannot reach the target network directly, but an agent positioned at the network boundary can.

Uses Low-Level Discovery (LLD) so that adding or removing monitored devices requires only a macro change — no edits to the agent configuration or manual item creation.

## Templates in this repository

| File | Template name | `{$PING_HOSTS}` format | Item / trigger names show |
|---|---|---|---|
| [`zbx_export_templates.yaml`](zbx_export_templates.yaml) | `Ping from host device` | `ip,ip,ip` | IP address |
| [`zbx_export_templates_hostname_ip.yaml`](zbx_export_templates_hostname_ip.yaml) | `Ping from host device - hostname and ip address` | `ip,hostname / ip,hostname` | Hostname and IP address |

Both templates work the same way; the second one additionally lets you give each device a name. See [Variant: hostname and IP address](#variant-hostname-and-ip-address) below.

> **Note:** Do not link both templates to the same host — they both create items with the key `ping.check[<ip>]`, so Zabbix will refuse to link the second one.

## How It Works

1. Zabbix Agent 2, running on a Windows host with access to the target network, is configured with two `UserParameter` scripts:
   - one that performs discovery, returning the list of IPs to monitor as LLD JSON
   - one that performs a single ping check against a given IP
2. The Zabbix server queries the discovery UserParameter, which reads the `{$PING_HOSTS}` macro (passed in from the template/host) and returns each IP as a `{#HOST_IP}` LLD macro.
3. Zabbix auto-generates one item, trigger, and graph per discovered IP, using the item prototypes defined on the template.
4. Each item independently runs a local ping check against its target IP and reports `1` (reachable) or `0` (unreachable).

## Installation

### 1. Configure Zabbix Agent 2

On the Windows host running Zabbix Agent 2, add the following `UserParameter` entries to `zabbix_agent2.conf` (or a file included via `Include=`):

```
UserParameter=ping.discovery[*],powershell -NoProfile -Command "$hosts = '$1'.Split(','); $out = @{data=@()}; foreach($h in $hosts){$out.data += @{'{#HOST_IP}'=$h.Trim()}}; $out | ConvertTo-Json -Depth 3 -Compress"

UserParameter=ping.check[*],powershell -NoProfile -Command "$target = '$1'; $t = \"Reply from $target`:\"; if(ping -n 1 -w 2000 $target | Select-String $t){1}else{0}"
```

Restart the Zabbix Agent 2 service for the changes to take effect.

### 2. Import and assign the template

Import the template into Zabbix and assign it to the host representing the Windows agent described above.

### 3. Configure macros

Set the following macros on the host (or override at host level if different values are needed per host):

| Macro | Description | Example |
|---|---|---|
| `{$PING_HOSTS}` | Comma-separated list of IP addresses to monitor | `10.0.5.10,10.0.5.11,10.0.5.12` |
| `{$TIME_OF_ONE_CHECK}` | Interval between individual ping checks | `1m` |
| `{$NUMBER_OF_CHECKS}` | Number of consecutive failed checks required before a device is considered unreachable and a trigger fires | `5` |

### 4. Run discovery

Wait for the discovery rule to run on its configured interval (default: `1h`), or trigger it manually via **Configuration > Hosts > Discovery**. Zabbix will automatically create one item, trigger, and graph per discovered IP address.

### 5. Maintain the device list

To add or remove monitored devices later, edit only the `{$PING_HOSTS}` macro. No changes are needed on the agent or Zabbix server side.

## Template Objects

### Discovery rule — `Create monitored ip addreses`

- **Key:** `ping.discovery["{$PING_HOSTS}"]`
- **Description:** Requests the list of IP addresses defined in `{$PING_HOSTS}` (set as a template or host macro) from the Zabbix Agent 2 and returns them as LLD data (`{#HOST_IP}`), enabling automatic creation of monitoring items for each address without manual configuration.
- **Timeout:** overridden to `20s`

> **Note:** The macro is wrapped in quotes (`["{$PING_HOSTS}"]`) so the entire comma-separated list is passed as a single parameter to the UserParameter, rather than being split into multiple positional parameters by Zabbix's own key parsing.

### Item prototype — `Try to ping {#HOST_IP} from {HOST.HOST}`

- **Key:** `ping.check[{#HOST_IP}]`
- **Description:** Executes a single ICMP ping to `{#HOST_IP}` from the agent host and returns `1` (reachable) or `0` (unreachable). Used to monitor availability of devices located in a network segment not directly accessible from the Zabbix server/proxy.
- **Update interval:** `{$TIME_OF_ONE_CHECK}`
- **Timeout:** overridden to `10s`

### Trigger prototype — `Device {#HOST_IP} is unreachable via ping from host {HOST.HOST}`

- **Expression:** `max(/Ping from host device/ping.check[{#HOST_IP}],{$NUMBER_OF_CHECKS})=0`
- **Severity:** Average
- **Description:** Fires when `{#HOST_IP}` has been unreachable via ping for the last `{$NUMBER_OF_CHECKS}` checks in a row (time to trigger: `{$NUMBER_OF_CHECKS} * {$TIME_OF_ONE_CHECK}`), indicating a sustained connectivity loss rather than a single failure.

### Graph prototype — `Avalability of {#HOST_IP} via ping`

Plots the ping reachability status (`1`/`0`) of `{#HOST_IP}` over time, used to visually track uptime and identify outage patterns.

## Notes

- Item and discovery rule timeouts are overridden above the 3s global default to accommodate PowerShell process startup overhead, which can otherwise cause `Timeout occurred while gathering data` errors.
- Requires Zabbix Agent 2 with Zabbix agent (passive) checks allowed from the server/proxy to this host.
- The ping check matches only `Reply from <target IP>:` in the output, to avoid false positives from intermediate routers returning "Reply from" messages (e.g. "Destination host unreachable") for IPs that don't actually exist or respond.

## Variant: hostname and IP address

File: [`zbx_export_templates_hostname_ip.yaml`](zbx_export_templates_hostname_ip.yaml) — template `Ping from host device - hostname and ip address`.

Same as the template above, but `{$PING_HOSTS}` holds an IP address **and a hostname** for each device. Discovery returns both `{#HOST_IP}` and `{#HOST_NAME}`, so items, triggers and graphs are named after the device, e.g. `Try to ping Core-Switch 10.0.5.10`.

### 1. Configure Zabbix Agent 2

Add the following `UserParameter` entries to `zabbix_agent2.conf` (or a file included via `Include=`). `ping.check` is the same as in the original template — if it is already present, add only `ping.discoveryIpHost`. Each entry must stay on a single line.

```
UserParameter=ping.discoveryIpHost[*],powershell -NoProfile -Command "$out = @{data=@()}; foreach($e in ('$1' -split '/')){$p = $e -split ',',2; $ip = $p[0].Trim(); if($ip -eq ''){continue}; $n = $ip; if($p.Count -gt 1 -and $p[1].Trim() -ne ''){$n = $p[1].Trim()}; $out.data += @{'{#HOST_IP}'=$ip;'{#HOST_NAME}'=$n}}; $out | ConvertTo-Json -Depth 3 -Compress"

UserParameter=ping.check[*],powershell -NoProfile -Command "$target = '$1'; $t = \"Reply from $target`:\"; if(ping -n 1 -w 2000 $target | Select-String $t){1}else{0}"
```

Restart the Zabbix Agent 2 service (`Restart-Service "Zabbix Agent 2"` in an administrator PowerShell).

### 2. Import, assign and configure

Import `zbx_export_templates_hostname_ip.yaml`, assign the template to the host representing the Windows agent, and set `{$PING_HOSTS}`:

| Macro | Description | Example |
|---|---|---|
| `{$PING_HOSTS}` | Devices to monitor: `,` between IP and hostname, `/` between devices. Hostname is optional — if omitted, the IP is used as the name. | `10.0.5.10,Core-Switch / 10.0.5.11,Printer-2F / 10.0.5.12` |
| `{$TIME_OF_ONE_CHECK}` | Interval between individual ping checks | `1m` |
| `{$NUMBER_OF_CHECKS}` | Number of consecutive failed checks required before a device is considered unreachable | `5` |

> **Why `,` and `/`?** By default Zabbix Agent 2 rejects UserParameter arguments containing `` \ ' " ` * ? [ ] { } ~ $ ! & ; ( ) < > | # @ `` (unless `UnsafeUserParameters=1` is set). `,` and `/` are allowed, so this format works with default agent settings. For the same reason, hostnames must not contain these characters, or `/` and `,`.

Test the discovery directly on the agent host:

```
zabbix_agent2.exe -t "ping.discoveryIpHost[\"10.0.5.10,Core-Switch / 10.0.5.12\"]"
```

Expected output:

```
{"data":[{"{#HOST_IP}":"10.0.5.10","{#HOST_NAME}":"Core-Switch"},{"{#HOST_IP}":"10.0.5.12","{#HOST_NAME}":"10.0.5.12"}]}
```

### Template objects

| Object | Name | Key / expression |
|---|---|---|
| Discovery rule | `Create monitored ip addresses` | `ping.discoveryIpHost["{$PING_HOSTS}"]` |
| Item prototype | `Try to ping {#HOST_NAME} {#HOST_IP}` | `ping.check[{#HOST_IP}]` |
| Trigger prototype | `Device {#HOST_NAME} {#HOST_IP} is unreachable via ping from host {HOST.HOST}` | `max(/Ping from host device - hostname and ip address/ping.check[{#HOST_IP}],{$NUMBER_OF_CHECKS})=0` |
| Graph prototype | `Availability of {#HOST_NAME} {#HOST_IP} via ping` | — |

Item keys still use the IP address, so each item stays unique even if two devices share a name. `{HOST.HOST}` is left out of the item name because Zabbix 6.0+ no longer resolves host macros in item names; it is kept in the trigger name, where it works.
