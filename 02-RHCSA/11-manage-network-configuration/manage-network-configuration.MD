# Managing Network Configuration

This chapter teaches you to inspect and configure network interfaces, manage NetworkManager connection profiles, edit their configuration files, set hostnames, and test name resolution. I’ve skipped the quiz, guided exercises, and lab. The notes follow the Chapter 11 teaching sections in your RH199 course. `01-RH199_RHCSA_Rapid_Track_en_10.0.pdf`

## 1. Network devices and connection profiles

RHEL uses NetworkManager to manage network devices and their settings.

- A **device** is a physical or virtual network interface, such as `ens3`.
- A **connection** (also called a profile) holds settings for a device, such as its IP address, gateway, and DNS servers.
- One device can have multiple profiles, but only one profile for that device can be active at a time.
- Persistent connection profiles are stored in `/etc/NetworkManager/system-connections/` as `.nmconnection` files.

RHEL 10 change: ifcfg files and `/etc/sysconfig/network-scripts/` are no longer supported. NetworkManager uses INI-style keyfiles instead.

### Inspect devices and connections

```bash
nmcli device status
nmcli connection show
nmcli connection show --active
```

You can abbreviate `device` to `dev` and `connection` to `con`, provided the abbreviation is clear:

```bash
nmcli dev status
nmcli con show
```

In device output, `connected (externally)` means NetworkManager does not manage that device.

## 2. Create and activate a connection profile

Create a basic Ethernet profile:

```bash
nmcli connection add con-name static-ens3 type ethernet ifname ens3
```

For a static IPv4 profile, specify the address, prefix, and gateway:

```bash
nmcli connection add con-name static-ens3 type ethernet ifname ens3 \
  ipv4.method manual \
  ipv4.addresses 192.168.0.5/24 \
  ipv4.gateway 192.168.0.254
```

A prefix of `/24` is equivalent to subnet mask `255.255.255.0`.

To create a profile using DHCP, set its IPv4 method to `auto`:

```bash
nmcli connection add con-name dhcp-ens3 type ethernet ifname ens3 \
  ipv4.method auto
```

Activate a profile by its connection name:

```bash
nmcli connection up static-ens3
```

Disconnect a device and deactivate its current connection:

```bash
nmcli device disconnect ens3
```

`connection up` takes a profile name. `device disconnect` takes a device name.

## 3. Modify a connection profile

Display the settings stored for a profile:

```bash
nmcli connection show static-ens3
nmcli -p connection show static-ens3
nmcli -f ipv4.method,ipv4.addresses,ipv4.gateway connection show static-ens3
```

The `-p` option makes output easier to read. In detailed output, lowercase setting names are configured profile properties; uppercase values are runtime properties.

Change a static IPv4 profile:

```bash
nmcli connection modify static-ens3 \
  ipv4.method manual \
  ipv4.addresses 192.0.2.2/24 \
  ipv4.gateway 192.0.2.254 \
  connection.autoconnect yes
```

To switch a profile from DHCP to static addressing, set `ipv4.method manual`. For IPv6, set the appropriate `ipv6.method`. If the method does not match your intended configuration, the connection may still use automatically obtained addresses or may not activate as expected.

Example IPv6 settings:

```bash
nmcli connection modify static-ens3 \
  ipv6.method manual \
  ipv6.addresses '2001:db8:0:1::a/64' \
  ipv6.gateway '2001:db8:0:1::1'
```

IPv6 methods to recognize:

- `manual`: static IPv6 configuration.
- `auto`: uses router advertisements and SLAAC.
- `dhcp`: uses DHCPv6, but not SLAAC.

### Add or replace list values

Some settings, such as DNS servers, can contain multiple values.

```bash
nmcli connection modify static-ens3 ipv4.dns 192.0.2.53
```

Without a prefix, this replaces the existing DNS list.

```bash
nmcli connection modify static-ens3 +ipv4.dns 192.0.2.54
nmcli connection modify static-ens3 -ipv4.dns 192.0.2.54
```

`+` adds a value; `-` removes one.

## 4. Verify current network settings

`nmcli connection show` shows the configured profile. The `ip` command shows the current runtime state.

```bash
ip -4 -br address show dev ens3
ip -6 -br address show dev ens3
```

This distinction is useful when diagnosing a change that is saved in a profile but not active on the interface yet.

## 5. Edit NetworkManager keyfiles

You can edit a profile directly, although `nmcli` is the preferred method for managing profiles.

A keyfile uses INI-style sections and key-value settings. For example:

```ini
[connection]
id=static-ens3
type=ethernet
autoconnect=true
interface-name=ens3

[ipv4]
method=manual
address1=172.25.0.10/24
gateway=172.25.0.254
dns=172.25.254.254;
dns-search=example.com
```

Useful keyfile-to-`nmcli` mappings:

| `nmcli` property | Keyfile section and setting | Purpose |
| --- | --- | --- |
| `ipv4.method manual` | `[ipv4] method=manual` | Static IPv4 settings |
| `ipv4.method auto` | `[ipv4] method=auto` | DHCPv4 |
| `ipv4.addresses 192.0.2.1/24` | `address1=192.0.2.1/24` | IPv4 address and prefix |
| `ipv4.gateway 192.0.2.254` | `gateway=192.0.2.254` | Default gateway |
| `ipv4.dns 8.8.8.8` | `dns=8.8.8.8;` | DNS server |
| `ipv4.dns-search example.com` | `dns-search=example.com` | Search domain |
| `ipv4.ignore-auto-dns yes` | `ignore-auto-dns=true` | Ignore DHCP-provided DNS |
| `ipv6.method manual` | `[ipv6] method=manual` | Static IPv6 settings |
| `ipv6.method auto` | `[ipv6] method=auto` | SLAAC/router advertisements |
| `ipv6.method dhcp` | `[ipv6] method=dhcp` | DHCPv6, without SLAAC |
| `ipv6.addresses 2001:db8::a/64` | `address1=2001:db8::a/64` | IPv6 address and prefix |
| `ipv6.gateway 2001:db8::1` | `gateway=2001:db8::1` | IPv6 default gateway |
| `ipv6.dns ...` | `[ipv6] dns=...` | IPv6 DNS server |
| `connection.autoconnect yes` | `[connection] autoconnect=true` | Activate this profile automatically |
| `connection.id ens3` | `[connection] id=ens3` | Profile name |
| `connection.interface-name ens3` | `[connection] interface-name=ens3` | Bind profile to interface |

Multiple addresses use incrementing keys such as `address1`, `address2`, and `address3`.

### Profile file locations

- `/etc/NetworkManager/system-connections/`: persistent profiles users create or modify.
- `/run/NetworkManager/system-connections/`: temporary profiles; removed after reboot.
- `/usr/lib/NetworkManager/system-connections/`: preconfigured, immutable profiles.

After manually editing a profile, reload NetworkManager’s profile data:

```bash
nmcli connection reload
```

Or reload only one profile:

```bash
nmcli connection reload static-ens3
```

Then activate it to apply its settings to the interface:

```bash
nmcli connection up static-ens3
```

Manual file edits need both steps: reload, then activate.

## 6. Delete a connection

```bash
nmcli connection delete static-ens3
```

This removes the connection profile and disconnects its device.

## 7. Permissions for network changes

Check your current NetworkManager permissions:

```bash
nmcli general permissions
```

Root has the permissions needed to manage network settings. Console users may have permission to manage connections, while users connected through SSH may need to become root or use `sudo`.

Be cautious when changing the profile for the interface carrying your SSH session: disconnecting or reactivating it can interrupt your connection.

## 8. Set the system hostname

A fully qualified domain name (FQDN) identifies a host using both its host name and domain, such as `servera.lab.example.com`.

### Temporary hostname

```bash
hostname web1.example.com
```

This changes the hostname for the current run; it does not persist after reboot.

### Persistent hostname

```bash
hostnamectl hostname db1.example.com
```

This updates the persistent hostname, stored in `/etc/hostname`. Check it with:

```bash
hostnamectl hostname
cat /etc/hostname
hostnamectl
```

The `hostnamectl` summary can show a static hostname and a transient hostname. If `/etc/hostname` is absent or empty, the system can derive a transient hostname from a reverse DNS lookup of the interface IP.

After changing the hostname, the current shell prompt might still show the old name because it inherited its environment when it started. Open a new terminal session to see the updated prompt.

## 9. Name resolution: `/etc/hosts`, DNS, and `/etc/resolv.conf`

When an application needs to resolve a hostname, the system uses a stub resolver to handle the lookup. The lookup order is controlled by `/etc/nsswitch.conf`. By default, the system checks `/etc/hosts` first.

Example `/etc/hosts` entry:

```text
172.25.250.10 servera.lab.example.com servera
```

This maps an IP address to a full host name and an optional short alias. If no matching entry exists, the resolver queries DNS.

`/etc/resolv.conf` controls DNS search and server settings:

```text
search lab.example.com
nameserver 172.25.250.100
```

- `search`: domain suffixes to try with short host names.
- `nameserver`: DNS server IP address. Up to three `nameserver` lines can be used.

NetworkManager commonly generates `/etc/resolv.conf` from DNS settings in the active connection profile. Configure DNS through the profile:

```bash
nmcli connection modify static-ens3 ipv4.dns 172.25.250.220
nmcli connection up static-ens3
```

To append rather than replace a DNS server:

```bash
nmcli connection modify static-ens3 +ipv4.dns 172.25.254.254
```

For a DHCP connection, NetworkManager can rewrite DNS settings from DHCP. To ignore those automatic DNS settings:

```bash
nmcli connection modify static-ens3 ipv4.ignore-auto-dns yes
```

For dual-stack networking, the course recommends ensuring there is a reachable DNS server over IPv4 and one over IPv6.

## 10. Test name resolution

```bash
host servera.lab.example.com
host 172.25.250.10
```

`host` tests DNS lookups, including reverse lookups for an IP address. `dig` provides more detailed DNS query output:

```bash
dig servera.lab.example.com
```

Important distinction: `host` and `dig` query DNS directly; they do not use `/etc/hosts`. To test resolution as applications on the system see it—including the normal `/etc/hosts`-first lookup order—use:

```bash
getent hosts servera.lab.example.com
```

## Exam workflow

For a static IPv4 network task:

```bash
nmcli dev status
nmcli con show
nmcli con add con-name static-ens3 type ethernet ifname ens3 \
  ipv4.method manual ipv4.addresses 172.25.250.10/24 \
  ipv4.gateway 172.25.250.254
nmcli con mod static-ens3 ipv4.dns 172.25.254.254
nmcli con mod static-ens3 connection.autoconnect yes
nmcli con up static-ens3
```

Then verify both profile and runtime state:

```bash
nmcli con show static-ens3
ip -4 -br address show dev ens3
```

If asked to set a hostname and verify name resolution:

```bash
hostnamectl hostname servera.lab.example.com
getent hosts servera.lab.example.com
host servera.lab.example.com
```

## Key distinctions to remember

- Device = interface; connection = configuration profile.
- `nmcli con up PROFILE` activates a profile; `nmcli dev disconnect DEVICE` disconnects an interface.
- `nmcli` profile settings are persistent; `ip` shows the current runtime state.
- Manual keyfile edits require `nmcli con reload` and then `nmcli con up PROFILE`.
- `hostname` changes are temporary; `hostnamectl hostname` makes the change persistent.
- `host` and `dig` query DNS; `getent hosts` follows the system’s normal lookup order.
