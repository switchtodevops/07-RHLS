# Registering Systems for Red Hat Support

Study notes for RHEL system registration. Commands below need root. Behavior matches current Red Hat documentation for RHEL 8.8 and later, RHEL 9, and RHEL 10, including Simple Content Access.

## Why register

Red Hat delivers RHEL packages and updates through the **Content Delivery Network (CDN)**. Those repositories include security fixes and bug fixes. An unregistered system cannot use the **CDN repositories** that require a subscription.

Registration also ***creates the system's identity in Red Hat's hosted services***. Depending on the account and the features you enable, that identity is what the Hybrid Cloud Console, Red Hat Lightspeed, knowledgebase, and support tools use to recognize the system.

Red Hat Lightspeed is the current name of the service formerly called Red Hat Insights. The client command is still `insights-client`.

Registration uses the organization's subscriptions. It does not create a new subscription, and it does not grant products the organization has not purchased.

## What registration does

1. The system authenticates with a Red Hat username and password, or with an activation key plus the organization's numeric organization ID.
2. Red Hat assigns the system a unique identity and stores the identity certificate on the system.
3. The system uses that identity to reach the repositories and services the organization is allowed to use.

Simple Content Access decides *how* content is authorized. Registration is still required either way.

## Simple Content Access

Simple Content Access (SCA) is the current subscription mode. New Red Hat accounts use it by default. The older entitlement mode, where each system had a subscription attached before it could use content, is deprecated.

Under SCA:

- The organization needs at least one valid subscription that covers the content.
- You register the system. You do not run `subscription-manager attach` or `--auto-attach` for each host.
- Subscription usage is reviewed in the Hybrid Cloud Console, at the organization level.
- The repositories that match the installed product are enabled automatically. Further repositories are enabled with `subscription-manager repos`.

```bash
subscription-manager repos --list-enabled
subscription-manager repos --enable <repo_id>
```

`subscription-manager` writes the enabled CDN repositories into `/etc/yum.repos.d/redhat.repo`. `dnf` then installs from that file.

## Which command to use

| Situation | Command |
| --- | --- |
| RHEL 8.8 or later, including RHEL 9 and RHEL 10. Preferred method. Also enables Lightspeed. | `rhc connect` |
| Content and repositories only, or any host where `rhc` is not the right client (including RHEL 8.7 and earlier, and Satellite-managed hosts) | `subscription-manager register` |
| Lightspeed on a system already registered with `subscription-manager` | `insights-client --register` |
| Graphical registration on an installed system | RHEL web console (Cockpit) |
| Registration while the OS is installing | Anaconda, or a Kickstart file with an activation key |

`rhc` is the client for remote host configuration. On RHEL 8 and 9, `rhc connect` turns on content, Lightspeed analytics, and remote management together. Choosing individual features with `--disable-feature` is a RHEL 10 option.

On a minimal install, install the client you need first: `dnf install rhc` or `dnf install insights-client`. `subscription-manager` is part of a normal RHEL install.

## Register with `rhc`

Interactive login:

```bash
rhc connect
```

The command prompts for the Red Hat username and password. You can also pass `--username` and `--password`.

With an activation key:

```bash
rhc connect --activation-key=<key_name> --organization=<org_ID>
```

A default connection enables three features:

| Feature | What it does |
| --- | --- |
| `content` | CDN repository access. Generates the redhat.repo file. |
| `analytics` | Registers the host with Red Hat Lightspeed and collects analytics data. Requires `content`. |
| `remote-management` | Starts the `yggdrasil` service so the Hybrid Cloud Console can run remediations and tasks. Requires `content` and `analytics`. On RHEL 10, install `rhc-worker-playbook` before those playbooks can run. |

Check the result:

```bash
rhc status
```

A full connection reports Subscription Management, Lightspeed, and an active `yggdrasil` service.

### RHEL 10 feature selection

On RHEL 10, disable a feature at connect time. The flag can be repeated.

Content and Lightspeed, without remote management:

```bash
rhc connect --activation-key=<key_name> --organization=<org_ID> \
  --disable-feature remote-management
```

Content only. Remote management depends on analytics, so both are disabled:

```bash
rhc connect --activation-key=<key_name> --organization=<org_ID> \
  --disable-feature analytics \
  --disable-feature remote-management
```

`--disable-feature=analytics,remote-management` is accepted by current `rhc` because the option is a list. Repeating the flag matches the documented form.

On RHEL 8 and 9, leave those flags off. For repositories without Lightspeed on those versions, register with `subscription-manager` and do not run `rhc connect` or `insights-client --register`.

## Activation keys

An organization administrator creates an activation key in the Hybrid Cloud Console. The key is a token for that organization. Pair it with the numeric organization ID. The ID is on the Activation Keys page in the console.

Use a key when a person will not be at the console, and in Kickstart files. The key keeps a personal username and password out of scripts.

An administrator can preconfigure the key with the RHEL release, system purpose, and extra repositories. Those settings are applied during registration.

`subscription-manager` accepts the same key. Current Red Hat docs spell the options as `--activation-key` and `--organization`. The older spellings `--activationkey` and `--org` are the same options:

```bash
subscription-manager register --activation-key=<key_name> --organization=<org_ID>
```

## Register with `subscription-manager`

Interactive:

```bash
subscription-manager register
```

Activation key form is in the previous section. Non-interactive username login:

```bash
subscription-manager register --username=<username> --password=<password>
```

This is valid on RHEL 10. With SCA you still do not attach a subscription afterward.

If this host should also report to Lightspeed, register that client after subscription-manager succeeds. `insights-client` reuses the identity certificate, so it does not ask for the password again:

```bash
insights-client --register
insights-client --status
```

`rhc connect` already does this Lightspeed registration when the `analytics` feature is left on.

## Confirm that registration worked

`subscription-manager identity` prints the system UUID, name, organization name, and organization ID after a successful registration. An unregistered system tells you to register.

```bash
subscription-manager identity
subscription-manager status
subscription-manager repos --list-enabled
```

`subscription-manager status` depends on the version and the account mode:

| System | Successful registration |
| --- | --- |
| Current RHEL 10, and updated RHEL 8.10 / RHEL 9 subscription-manager | `Overall Status: Registered` |
| Older subscription-manager with SCA | `Overall Status: Disabled`, plus a line that content access mode is Simple Content Access. The entitlement check is turned off because SCA does not attach subscriptions. The host is registered. |
| Deprecated entitlement mode, with a covering subscription attached | `Overall Status: Current` |
| Not registered | `Overall Status: Unknown`, or a message that the system is not registered |

On an SCA account, `Disabled` on an older build means the per-system entitlement calculation is off. Confirm the identity with `subscription-manager identity`.

After an `rhc` connection, also run `rhc status`.

## System purpose

System purpose is inventory data. It records how the organization intends to use the system so the Hybrid Cloud Console can filter and report usage. It is optional, and it is separate from registration.

Set it on an activation key, in the installer, in Kickstart, or after installation. Under SCA the purpose *status* is disabled, because purpose is no longer used to pick a subscription to attach. The values are still worth setting for reporting.

Default values:

| Attribute | Values | Meaning |
| --- | --- | --- |
| Role | `Red Hat Enterprise Linux Server`, `Red Hat Enterprise Linux Workstation`, `Red Hat Enterprise Linux Compute Node` | Workload the system runs |
| Service level | `Premium`, `Standard`, `Self-Support` | Support level expected for this system |
| Usage | `Production`, `Development/Test`, `Disaster Recovery` | Environment |

The role string is the full product name. `Workstation` alone is not the value the command stores.

```bash
subscription-manager syspurpose role --set "Red Hat Enterprise Linux Workstation"
subscription-manager syspurpose service-level --set "Premium"
subscription-manager syspurpose usage --set "Development/Test"

subscription-manager syspurpose role --list
subscription-manager syspurpose service-level --list
subscription-manager syspurpose usage --list
subscription-manager syspurpose --show
```

`--list` shows the values this organization's subscriptions allow. Installed products can add more roles than the three defaults.

## Other registration paths

**Web console.** Install Cockpit and the subscriptions page (`subscription-manager-cockpit`). Open the web console, go to **Subscriptions**, and register with an account or an activation key.

**Interactive install.** Anaconda has a **Connect to Red Hat** screen. It accepts an account or an activation key, and it can set system purpose before the install finishes.

**Kickstart.** Put the organization ID and activation key in the file:

```bash
rhsm --organization=<org_ID> --activation-key=<key_name>
syspurpose --role="Red Hat Enterprise Linux Server" --sla="Premium" --usage="Production"
```

## Unregister

Use the client that created the connection.

```bash
rhc disconnect
```

`rhc disconnect` removes the subscription-manager registration, disconnects Lightspeed, and stops remote management.

```bash
subscription-manager unregister
```

`subscription-manager unregister` deletes the local identity and entitlement certificates and removes the system from subscription management. A host that was also connected with `rhc` should be disconnected with `rhc disconnect` so Lightspeed and `yggdrasil` are removed with it.

Lightspeed alone:

```bash
insights-client --unregister
```

That stops analytics uploads. It leaves the subscription-manager registration in place.

## Certificates

Registration stores X.509 certificates under `/etc/pki`. `subscription-manager unregister` and `rhc disconnect` remove the identity and entitlement certificates.

| Path | Role |
| --- | --- |
| `/etc/pki/consumer/` | System identity. `cert.pem` is the consumer certificate and `key.pem` is its private key. |
| `/etc/pki/entitlement/` | Certificates that authenticate the system to CDN repositories. With SCA this is a content-access certificate. In entitlement mode these are the attached-subscription certificates. |
| `/etc/pki/product/` | Certificates for installed Red Hat products. |
| `/etc/pki/product-default/` | Default product certificates shipped with the installation. They identify the product so the matching base repositories can be enabled. |

Read a certificate:

```bash
rct cat-cert /etc/pki/consumer/cert.pem
```

`rct` comes with `subscription-manager`. `rct stat-cert <file>` prints a short summary.

## Exam points

- Registration gives the system a unique Red Hat identity and access to entitled CDN content.
- You need root, plus either account credentials or an activation key and the numeric organization ID.
- `rhc connect` is the preferred registration command on RHEL 8.8 and later. It also enables Lightspeed unless you are on RHEL 10 and disable `analytics`.
- `subscription-manager register` remains valid on RHEL 10. Use it for repository access without Lightspeed, and on RHEL 8.7 and earlier.
- Under SCA, registration is enough. Do not attach a subscription to each system. Enable any extra repositories explicitly.
- Prove registration with `subscription-manager identity`. On current RHEL, `subscription-manager status` then shows `Overall Status: Registered`. On older SCA clients the same success is `Overall Status: Disabled`.
- System purpose uses the full role name, such as `Red Hat Enterprise Linux Server`, plus a service level and a usage value.
- `rhc disconnect` is the matching undo for `rhc connect`. `subscription-manager unregister` is the matching undo for `subscription-manager register`.
