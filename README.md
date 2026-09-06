# ansible_role_sandboxing

An Ansible role that manages Linux **sandbox users** on a system and their
network isolation. It creates unprivileged users, provisions one TAP interface
per sandbox, and enforces firewalld/nftables forwarding policy so each sandbox
is isolated from the host and private networks.

When the configured `sandboxing_count` is lowered, the role automatically
cleans up the now-unused managed users, homes, network units, interfaces,
firewalld policies/zone and `/etc/hosts` entries.

## Requirements

- Controller (for `ansible.posix`): `ansible.posix` collection

  ```sh
  ansible-galaxy collection install -r requirements.yml
  ```

- Target host:
  - Debian 12+ (bookworm/trixie) or Ubuntu 22.04+ (jammy/noble) with systemd
  - `systemd-networkd`
  - `firewalld` (uses the `nftables` backend)
  - a `kvm` group (usually provided by `udev`)
  - a single default IPv4 route (the role masquerades that interface)

The role installs `iproute2`, `nftables` and `firewalld`, unmask/enable/start
`systemd-networkd`, and requires root privileges.

## Role Variables

All variable names are prefixed with `sandboxing_`.

| Variable | Default | Description |
| --- | --- | --- |

> **Warning:** `sandboxing_count` defaults to `0`. The role is **not**
> additive-only: the cleanup steps remove every resource the role manages
> (users, homes, TAPs, networkd units, firewalld policies/zone, `/etc/hosts`
> entries) whose index is `>= sandboxing_count`. Running the role with the
> default `0` — or lowering the count — therefore **deletes everything** the
> role previously created. Set `sandboxing_count` to the number of sandboxes
> you actually want before running.

| Variable | Default | Description |
| --- | --- | --- |
| `sandboxing_count` | `0` | Number of sandboxes to create. Sandboxes are named `sandboxing_base_name` + index (`0`..`count-1`). Lowering this value triggers cleanup of the higher-indexed (now unused) sandboxes. |
| `sandboxing_network_base_octets` | `192.168.255` | First three octets of the addressing space (the fourth octet is allocated per sandbox). The allocated space must stay inside the same `/16` as the base. |
| `sandboxing_base_home` | `/opt/sandboxing` | Base folder holding each sandbox's home. |
| `sandboxing_base_name` | `sandbox` | Base name shared by sandbox users/groups/TAPs. |
| `sandboxing_network_prefix` | `/30` | Subnet prefix length for each sandbox. |
| `sandboxing_network_size` | `4` | Hosts per sandbox subnet (must be `>= 4`, consistent with the prefix). |
| `sandboxing_firewalld_default_zone` | `sandboxing` | Name of the dedicated sandbox firewalld zone. |

### What gets created per sandbox `i`

- User/group `sandbox<i>` with a locked password, `0700` home,
  empty home contents, and membership in the `kvm` supplementary group.
- A TAP interface `sandbox<i>` owned by that user, via `systemd-networkd`.
- A /30 subnet starting at `sandboxing_network_base_octets` + `i * 4`
  (the TAP address is the first usable address of the subnet).
- An `/etc/hosts` entry mapping the sandbox's VM IP to `sandbox<i>`, and
  an entry mapping the subnet gateway to `gateway_sandbox<i>`.

### Networking / firewall summary

- A global firewalld config enforces the `nftables` backend.
- The default-route interface is bound to the `external` zone for NAT
  masquerading.
- A dedicated `sandboxing` zone is provisioned only while at least one sandbox
  is configured.
- Forwarding is allowed from the sandbox zone to any zone by default, but:
  - forwarded traffic to non-public IPv4 networks (RFC 1918, link-local,
    CGNAT, TEST-NET, multicast, etc.) is rejected,
  - sandbox-initiated traffic to the host is rejected.
- IPv4 forwarding is enabled (IPv6 and broadcast forwarding disabled) via a
  dedicated sysctl file.

## Dependencies

- Collection: `ansible.posix` (uses `ansible.posix.firewalld` and
  `ansible.posix.sysctl`).

## Example Playbook

```yaml
- hosts: sandbox-hosts
  become: true
  vars:
    sandboxing_count: 4
    sandboxing_network_base_octets: 10.42.0
  roles:
    - role: ansible_role_sandboxing
```

## Testing

Testing is done with `tox` (python version and environment manager),
`molecule` (ansible role testing framework) and `podman` (target
container system-under-test).

For latest ansible version, the configuration uses python 3.13 (debian 13).
To test with the latest version of ansible :

```sh
tox -e latest
```

For earliest version, the configuration requires python 3.11
(install it using `asdf`, or any other python manager).
To test with the minimal version of ansible :

```sh
tox -e minimal # UNTESTED
```

See `tox.ini` for more details.

## License

MIT

## Author Information

nipil
