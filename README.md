# WireGuard

A fast, simple VPN library for Arduino on Zephyr. Much smaller than TLS-based
alternatives.

## Setup

- Install [ArduinoCore-Zephyr](https://github.com/arduino/ArduinoCore-zephyr)
- There is an official PR pending, setup the [fork](https://github.com/gibix/zephyr/tree/zephyr-arduino-wireguard)

Add this to you board variant `.conf`

```txt
CONFIG_WIREGUARD=y
CONFIG_WIREGUARD_MAX_PEER=1
CONFIG_WIREGUARD_MAX_SRC_IPS=2
CONFIG_NET_IF_MAX_IPV4_COUNT=3
CONFIG_NET_IF_UNICAST_IPV4_ADDR_COUNT=2
CONFIG_NET_RX_STACK_SIZE=2560
```

Add exported symbols

```
#if defined(CONFIG_NETWORKING)
FORCE_EXPORT_SYM(net_if_foreach);
FORCE_EXPORT_SYM(net_if_down);
FORCE_EXPORT_SYM(net_if_up);
FORCE_EXPORT_SYM(net_if_get_by_iface);
#if defined(CONFIG_NET_IPV4)
FORCE_EXPORT_SYM(net_if_ipv4_maddr_add);
FORCE_EXPORT_SYM(net_if_ipv4_maddr_join);
FORCE_EXPORT_SYM(net_if_ipv4_set_gw);
FORCE_EXPORT_SYM(net_if_ipv4_addr_add);
FORCE_EXPORT_SYM(net_if_ipv4_set_netmask);
FORCE_EXPORT_SYM(net_if_ipv4_set_netmask_by_addr);
#endif
FORCE_EXPORT_SYM(net_if_lookup_by_dev);
FORCE_EXPORT_SYM(net_ipaddr_parse);
FORCE_EXPORT_SYM(net_ipaddr_parse_mask);
FORCE_EXPORT_SYM(net_mask_len_to_netmask);
#endif

#if defined(CONFIG_NET_L2_VIRTUAL)
FORCE_EXPORT_SYM(_net_l2_VIRTUAL);
FORCE_EXPORT_SYM(net_mgmt_NET_REQUEST_VIRTUAL_INTERFACE_SET_PRIVATE_KEY);
#endif

#if defined(CONFIG_WIREGUARD)
FORCE_EXPORT_SYM(wireguard_peer_add);
FORCE_EXPORT_SYM(wireguard_peer_remove);
FORCE_EXPORT_SYM(wireguard_peer_keepalive);
#endif
```

## Why

Classic VPN uses:

- Private overlay network between devices
- Route traffic through a remote server
- Bypass censorship or geo-restrictions

Extra benefits over TLS:

- Mutual authentication built in (no need for mTLS)
- Works at L3: secures TCP, UDP, and any other protocol
- Native roaming: switch WiFi networks without dropping the tunnel
- Tiny footprint (~600 B vs ~8-15 KB RAM for TLS)
- Compatible with most commercial VPN services

## Use Cases

- Encrypted peer-to-peer link over IPv4 or IPv6
- Overlay network for distributed IoT devices
- NAT traversal without port forwarding
- Roaming: move between networks, keep the same tunnel address
- Secure remote access to home or office devices

## WireGuard and TLS

Compared to TLS (using `ZephyrSSLClient` with mbedTLS), WireGuard has a much
smaller RAM footprint at the sketch level:

| | WireGuard | TLS |
|--|-----------|-----|
| Static globals | ~37 B | ~2,040 B |
| Runtime heap | 0 | ~2 KB |
| Subsystem state | kernel-managed | ~4-6.5 KB (mbedTLS) |
| **Total** | **~600 B** | **~8-11 KB** |
| **Peak (handshake)** | **~600 B** | **~10-15 KB** |

The TLS cost is dominated by mbedTLS structures (`ssl_context`, `ssl_config`,
`x509_crt`) and the embedded PEM CA certificate (~1.9 KB). WireGuard's crypto
state is managed entirely by the Zephyr kernel networking layer — the sketch
only holds a lightweight handle.

WireGuard also operates at L3, so it secures all traffic (TCP, UDP, etc.)
without per-connection overhead, while TLS requires a separate handshake and
context for each TCP connection.
