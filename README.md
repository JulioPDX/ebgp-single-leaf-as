# DC1 AVD fabric

AVD 6.4.0 managed with uv: two spines and four independent L3 leaves. eBGP carries loopback and VTEP reachability in the underlay; eBGP EVPN provides the VXLAN overlay control plane.

Both spines use ASN 65100 and act as EVPN route servers. Leaves 1 and 2 (`dc1-leaf1a`/`dc1-leaf1b`) use ASN 65101; leaves 3 and 4 (`dc1-leaf2a`/`dc1-leaf2b`) use ASN 65102. MLAG is disabled, and every leaf has its own VTEP address. Both leaf BGP peer groups use `allowas-in 1` to accept routes from the other leaf sharing their ASN through either spine. Leaf gateways use anycast addressing when tenant services are added.

| Node | ASN | Router-ID / Loopback0 | VTEP / Loopback1 |
| --- | --- | --- | --- |
| dc1-spine1 | 65100 | 10.255.0.1 | — |
| dc1-spine2 | 65100 | 10.255.0.2 | — |
| dc1-leaf1a (leaf 1) | 65101 | 10.255.0.3 | 10.255.1.3 |
| dc1-leaf1b (leaf 2) | 65101 | 10.255.0.4 | 10.255.1.4 |
| dc1-leaf2a (leaf 3) | 65102 | 10.255.0.5 | 10.255.1.5 |
| dc1-leaf2b (leaf 4) | 65102 | 10.255.0.6 | 10.255.1.6 |

All six nodes use the `cEOS` platform with `Management0` for container management.

## Environment

Run commands from the project root. Python 3.12 or 3.13 and uv are required.

```sh
export UV_CACHE_DIR="$PWD/.cache/uv"
uv sync --locked
uv run ansible-galaxy collection install -r requirements.yml -p .ansible/collections
uv run python -c 'import pyavd; print(pyavd.__version__)'
uv run ansible-galaxy collection list
```

The Python package and Ansible collection are both pinned to 6.4.0, with Ansible Core pinned to the compatible 2.20.4 release. The environment uses Python 3.12. Local collection and cache directories are ignored by Git.

## Build

```sh
export UV_CACHE_DIR="$PWD/.cache/uv"
uv run ansible-inventory --graph
uv run ansible-playbook build.yml
```

The default inventory is [sites/dc1/inventory.yml](sites/dc1/inventory.yml). Generated EOS configurations are in `intended/configs/`, structured configurations in `intended/structured_configs/`, and fabric/device documentation in `documentation/`. Shared `root_dir` settings keep these outputs at the project root for all playbooks.

Fabric routing settings are under `sites/dc1/group_vars/FABRIC/`; node definitions are under `DC1_SPINES/` and `DC1_L3_LEAFS/`. Add tenants to `NETWORK_SERVICES/network_services.yml` and server adapters to `CONNECTED_ENDPOINTS/connected_endpoints.yml`.

The complete build passed merged-input and generated-configuration schema validation for all six devices. Artifact review confirmed eight reciprocal eBGP underlay peerings, eight EVPN peerings, unique router IDs and VTEPs, and `allowas-in 1` on both leaf peer groups. No live devices have been deployed or tested. VXLAN source interfaces are configured; VLAN/VNI mappings will be generated when tenant services are added.

## Cabling

| Leaf | Leaf Ethernet1 → spine1 | Leaf Ethernet2 → spine2 |
| --- | --- | --- |
| dc1-leaf1a | Ethernet1 | Ethernet1 |
| dc1-leaf1b | Ethernet2 | Ethernet2 |
| dc1-leaf2a | Ethernet3 | Ethernet3 |
| dc1-leaf2b | Ethernet4 | Ethernet4 |

Leaf Ethernet3 and higher remain available for endpoints. No leaf-to-leaf peer links are required.

## Address pools

| Purpose | Pool |
| --- | --- |
| Router-ID / EVPN Loopback0 | 10.255.0.0/27 (leaf offset 2) |
| Unique VTEP Loopback1 | 10.255.1.0/27 (leaf offset 2) |
| Leaf–spine routed links | 10.255.255.0/27 |
