# Managing Network Security

This chapter covers two layers of network security:

1. Firewalld controls which network traffic can reach a system’s services.
2. SELinux port labeling controls which services are allowed to listen on particular ports.

Both layers may need to permit a connection. Opening a port in the firewall does not override an SELinux restriction.

## 19.1 Managing Server Firewalls

### Linux firewall architecture

The kernel provides netfilter, which handles network traffic operations such as packet filtering and network address or port translation. nftables builds on netfilter to apply firewall rules. In RHEL 10, nftables is the firewall framework.

firewalld is the recommended dynamic firewall manager and the front end for nftables. Administrators usually manage it with:

- `firewall-cmd` from the command line
- The Firewall page in the web console

Older `iptables` and `ip6tables` tools have been succeeded by nftables. The chapter mentions `iptables-translate` and `ip6tables-translate` for converting older configurations.

### How firewalld zones work

Firewalld groups firewall rules into zones. A packet’s zone is chosen according to its source address or incoming network interface:

1. If the source address belongs to a zone, that zone’s rules apply.
2. Otherwise, the zone associated with the incoming interface applies.
3. If neither is associated with a zone, the default zone applies.

The default zone is one of the existing zones; it is not a separate zone. Initially, `public` is the default zone, and the loopback interface `lo` is assigned to `trusted`. NetworkManager can automatically set a zone for a connection, which is useful when a system moves between networks.

Zones generally allow traffic for established connections initiated by the system and allow outgoing traffic. They allow incoming connections that match an enabled service or port. Unmatched incoming traffic is usually rejected. `trusted` is the exception: it allows all incoming traffic by default.

### Predefined zones

Zones are predefined and can be customized.

| Zone | Initial behavior |
| --- | --- |
| `trusted` | Allows all incoming traffic |
| `home` | Allows related traffic and `ssh`, `mdns`, `ipp-client`, `samba-client`, and `dhcpv6-client` |
| `internal` | Starts with the same allowed services as `home` |
| `work` | Allows related traffic and `ssh`, `ipp-client`, and `dhcpv6-client` |
| `public` | Allows related traffic and `ssh` and `dhcpv6-client`; default for newly added interfaces |
| `external` | Allows related traffic and `ssh`; masquerades forwarded IPv4 traffic |
| `dmz` | Allows related traffic and `ssh` |
| `block` | Rejects all incoming traffic unless it is related to outgoing traffic |
| `drop` | Drops incoming traffic unless it is related to outgoing traffic; does not send ICMP error responses |

Masquerading makes forwarded IPv4 traffic appear to originate from the outgoing network interface’s IPv4 address.

### Predefined services

A firewalld service is a named configuration that groups the ports and protocols required by a common service. Using a predefined service avoids having to determine and enter each port manually.

Examples from the chapter:

| Service | Traffic allowed |
| --- | --- |
| `ssh` | `22/tcp` |
| `dhcpv6-client` | `546/udp` on the `fe80::/64` IPv6 network |
| `ipp-client` | `631/udp` |
| `samba-client` | `137/udp` and `138/udp` |
| `mdns` | `5353/udp` to the IPv4 and IPv6 multicast addresses |
| `cockpit` | `9090/tcp` |

List the available predefined service names with:

```bash
firewall-cmd --get-services
```

If a predefined service does not fit your needs, allow specific ports and protocols instead.

### Manage the firewall from the web console

In the web console, sign in and enable administrative access using **Limited access** or **Turn on administrative access**, then enter your password. The console uses the user’s sudo configuration for elevated access.

Go to **Networking → Firewall → Edit rules and zones**. The Firewall page shows active zones and their allowed services. Expand a service to view details, or use **Add services** to allow a predefined service in a zone. The chapter advises returning to limited access when the administrative task is complete.

### `firewall-cmd` essentials

Most `firewall-cmd` commands change the runtime configuration unless you specify `--permanent`. A permanent change does not take effect in the current runtime configuration until you reload firewalld.

Use `--zone=ZONE` to choose a zone. If you omit it, the default zone is used. Use CIDR notation for networks, such as `192.168.1.0/24`.

| Command option | Purpose |
| --- | --- |
| `--get-default-zone` | Show the current default zone |
| `--set-default-zone=ZONE` | Set the default zone in both runtime and permanent configurations |
| `--get-zones` | List available zones |
| `--get-active-zones` | List zones in use and their interfaces or sources |
| `--add-source=CIDR` | Assign traffic from an address or network to a zone |
| `--remove-source=CIDR` | Remove a source assignment from a zone |
| `--add-interface=INTERFACE` | Assign an interface to a zone |
| `--change-interface=INTERFACE` | Change an interface’s zone |
| `--list-all` | List a zone’s interfaces, sources, services, and ports |
| `--list-all-zones` | List configuration for every zone |
| `--add-service=SERVICE` | Allow a predefined service |
| `--remove-service=SERVICE` | Remove an allowed service |
| `--add-port=PORT/PROTOCOL` | Allow a port, such as `8080/tcp` |
| `--remove-port=PORT/PROTOCOL` | Remove an allowed port |
| `--reload` | Replace runtime configuration with the permanent configuration |
| `--runtime-to-permanent` | Save the runtime rules as permanent rules |

### Runtime versus permanent

- Runtime changes apply immediately but are lost when firewalld restarts or the system reboots.
- Permanent changes persist but require `--reload` to apply them to runtime.
- `--reload` applies the permanent configuration and drops the current runtime configuration.
- `--runtime-to-permanent` saves current runtime rules as permanent rules.

### Examples

Set `dmz` as the default zone:

```bash
firewall-cmd --set-default-zone=dmz
```

Assign a network to `internal` and allow MySQL there, permanently:

```bash
firewall-cmd --permanent --zone=internal --add-source=192.168.0.0/24
firewall-cmd --permanent --zone=internal --add-service=mysql
firewall-cmd --reload
```

Assign one IPv4 address to `public`:

```bash
firewall-cmd --permanent --zone=public --add-source=172.25.25.11/32
firewall-cmd --reload
```

Use rich rules for more complex configurations. Direct configuration rules provide still more control; both are beyond the scope of this chapter.

## 19.3 SELinux Port Labeling

### Why SELinux labels ports

SELinux labels network ports with contexts and uses service policies to control which processes can bind to them. For example:

- SSH commonly uses port `22/tcp`, labeled `ssh_port_t`.
- HTTP commonly uses `80/tcp` and `443/tcp`, labeled `http_port_t`.

When a service tries to listen on a port, SELinux checks whether the service’s policy allows that process to bind to the port’s SELinux type. This helps stop an unauthorized service from taking over a port used by another legitimate service.

A nonstandard port may be blocked if it does not have an appropriate SELinux label, even when the firewall permits traffic to it. A port can have only one SELinux port context.

### Inspect port numbers and labels

Check the standard service-to-port information in `/etc/services`:

```bash
grep gopher /etc/services
```

List SELinux port label assignments:

```bash
semanage port -l
```

Filter by service-related text or by port number:

```bash
semanage port -l | grep ftp
semanage port -l | grep -w 70
```

A label can appear in separate entries for different protocols, such as TCP and UDP.

### Add, modify, and delete port labels

Use `semanage port` to manage port bindings. The options to know are:

- `-a` — add a port binding
- `-m` — modify a port binding
- `-d` — delete a port binding
- `-t TYPE` — specify the SELinux port type
- `-p PROTOCOL` — specify the protocol, such as `tcp` or `udp`
- `-l` — list port bindings
- `-C` — show local changes to the default policy

General pattern:

```bash
semanage port -a -t PORT_TYPE -p tcp|udp PORT_NUMBER
```

Example: label `71/tcp` for a Gopher service:

```bash
semanage port -a -t gopher_port_t -p tcp 71
```

Show local changes:

```bash
semanage port -l -C
```

Change the label on `71/tcp` to `http_port_t`:

```bash
semanage port -m -t http_port_t -p tcp 71
```

Delete the `gopher_port_t` binding for `71/tcp`:

```bash
semanage port -d -t gopher_port_t -p tcp 71
```

Check the resulting bindings:

```bash
semanage port -l -C
semanage port -l | grep http
```

Policy limitation: Most services provide an SELinux policy module with their expected port contexts. You cannot change a default port label supplied by a service’s policy using `semanage`; that requires modifying and reloading the targeted service policy module, which is outside this lesson.

Service-specific SELinux man pages use names such as `httpd_selinux(8)`. They describe relevant SELinux types, Booleans, and port types, and are not installed by default. Install the documentation package and search for them with:

```bash
dnf -y install selinux-policy-doc
man -k _selinux
```

## Exam checklist

- Firewall question: Identify the correct zone, then allow a predefined service or a specific `PORT/PROTOCOL`.
- Permanent firewall change: Include `--permanent`, then run `firewall-cmd --reload`.
- Runtime-only rule: Omit `--permanent`; it takes effect immediately but does not persist.
- Zone selection: Use `--zone=ZONE`; otherwise the command uses the default zone.
- SELinux blocks a service from binding: Check the port’s SELinux type with `semanage port -l`. Add or modify the binding to the type expected by the service.
- Keep the layers distinct: Firewalld governs network traffic reaching the host; SELinux governs whether a process may bind to a port.
- Know the `semanage port` flags: `-a`, `-m`, `-d`, `-t`, `-p`, and the inspection options `-l` and `-C`.

Chapter summary: Firewalld manages packet filtering and related network rules through zones, using `firewall-cmd` or the web console. SELinux protects service ports through port labels and targeted policy; use `semanage port` to inspect and manage permitted port bindings.
