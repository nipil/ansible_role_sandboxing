# AGENTS.md

Ansible role that manages Linux "sandbox" users on a system.

## Purposes

- Validate configuration and environment (see `validation.yml`).
- Install required packages (`iproute2`, `nftables`, `firewalld`) and ensure `systemd-networkd` is unmasked, enabled, and started.
- Deploy a global firewall config enforcing the `nftables` backend and restart firewalld (see `firewall.yml` and `files/firewalld.conf`).
- Bind the default-route interface to the `external` zone for masquerading, and provision a dedicated sandbox firewalld zone (when at least one sandbox is configured); the zone is removed when no sandboxes remain configured.
- Configure sandbox forwarding using firewalld policies (when at least one sandbox is configured): allow forwarded traffic from the sandbox zone to any zone by default, reject (for easier diagnostics) forwarded traffic destined for non-public IPv4 networks, and reject sandbox-initiated traffic to the host.
- Create a base folder for sandboxing users' homes.
- Create N sandbox users/groups (named base name + index), each with a locked password, a `0700` home directory, empty home contents, and membership in the `kvm` supplementary group.
- Create one TAP interface per sandbox using `systemd-networkd` units.
- Assign each TAP a sequential subnet (configurable size, default /30) starting at the network base octets, with TAP IP set to the first usable address of each subnet.
- Ensure each TAP interface is administratively up (checks state, brings up if needed).
- Bind each sandbox TAP to the sandbox firewalld zone.
- Add a per-sandbox `/etc/hosts` entry mapping a VM IP on the sandbox subnet to the sandbox name, and an entry mapping the gateway IP to a `gateway_`-prefixed sandbox name (see `hosts.yml`).
- Configure kernel forwarding via `sysctl` (in a dedicated sysctl file): enable IPv4 forwarding and disable IPv6 / IPv4 broadcast forwarding for the all and default scopes.
- Clean up any managed sandbox users/directories whose index is >= configured count.
- Clean up unused managed `systemd-networkd` unit files, network interfaces, and `/etc/hosts` entries for sandboxes whose index is >= configured count.
- Cleanup runs in a specific order: networks first, then interfaces, then sandbox firewalld policies and zone (removed when no sandboxes remain), then users, then homes, then hosts.

## Layout

- `tasks/main.yml` — entrypoint: validation → package/service/firewall setup → base folder → per-sandbox setup → cleanup (networks → interfaces → sandbox firewalld policies/zone → users → homes → hosts) → kernel forwarding.
- `tasks/sandbox.yml` — per-sandbox name validation, delegates to `user.yml`, `network.yml`, and `hosts.yml`.
- `tasks/user.yml` — creates group + user, secures home, empties home contents.
- `tasks/network.yml` — renders per-sandbox `.netdev` and `.network` units, binds the TAP into the sandbox firewalld zone, flushes handlers, checks interface state, brings up if needed.
- `tasks/firewall.yml` — deploys the firewalld config, starts firewalld, masquerades the default-route interface via the `external` zone, provisions the sandbox firewalld zone (when at least one sandbox is configured), deploys sandbox forwarding policy files, and asserts the `nftables` backend is active.
- `tasks/hosts.yml` — adds a per-sandbox `/etc/hosts` entry mapping for the sandbox and its gateway.
- `tasks/forwarding.yml` — sets network forwarding sysctls via `ansible.posix.sysctl`.
- `tasks/cleanup_networks.yml` — removes unused managed networkd files via `find` with anchored matching + Jinja2 regex index extraction.
- `tasks/cleanup_interfaces.yml` — finds managed network interfaces via `find` with anchored matching + Jinja2 regex index extraction, computes the unused list once, deletes those devices first, then removes their bindings from the sandbox firewalld zone.
- `tasks/cleanup_users.yml` — removes unused managed users via `getent`, fullmatch filtering, and Jinja2 regex index extraction.
- `tasks/cleanup_homes.yml` — removes unused managed home directories via `find` with anchored matching + Jinja2 regex index extraction.
- `tasks/cleanup_hosts.yml` — removes unused managed `/etc/hosts` entries via `grep` with anchored matching + Jinja2 regex index extraction.
- `tasks/validation.yml` — validates required role configuration, network base octet format/range, `/16` boundary safety for computed sandbox subnet addressing, and asserts the `kvm` group exists.
- `vars/main.yml` — role defaults and overridable configuration values.
- `defaults/main.yml` — default count and network base octets.
- `handlers/main.yml` — reloads `systemd-networkd` and reloads `firewalld` when managed units/zone bindings change.
- `files/firewalld.conf` — firewalld config enforcing the `nftables` backend.
- `templates/sandbox.netdev.j2` — TAP netdev with `Name` and `Kind=tap`, owned by sandbox user/group.
- `templates/sandbox.network.j2` — network unit matching sandbox name, assigning the sandbox address on its subnet.
- `templates/sandbox-to-any.policy.j2` — firewalld policy allowing sandbox forwarding to any zone while rejecting non-public IPv4 destinations.
- `templates/sandbox-to-host.policy.j2` — firewalld policy rejecting sandbox-initiated traffic to the host.
- `molecule/` — molecule test scaffolding for the role.

## Conventions

- Do not add comments to code unless asked.
- Match the existing Jinja2 / Ansible style used across the role.
