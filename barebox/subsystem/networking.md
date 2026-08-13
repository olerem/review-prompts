# Barebox Networking

Barebox does **not** have the Linux network stack. There is no `sk_buff`, no
NAPI, no netdev, no netfilter, no ethtool, no socket layer and no uAPI. Do not
load `../../kernel/subsystem/networking-core.md` or
`../../kernel/subsystem/networking-drivers.md`; every finding they generate
here is invented.

What exists is a poll-driven stack in `net/` with a driver interface in
`include/net.h`, sufficient for TFTP, NFS, DHCP, DNS, ping and fastboot.

## The driver interface

A driver fills a `struct eth_device` (`include/net.h`) and calls
`eth_register()`. The operations are:

| Op | Called when | Must do |
|----|-------------|---------|
| `init` | once, at register | one-time controller setup |
| `open` | interface goes up | start the MAC, connect the PHY |
| `send` | a frame is queued | transmit `packet`/`length`, synchronously |
| `recv` | every poll | drain the RX ring, call `net_receive()` per frame |
| `halt` | interface goes down | stop the MAC and DMA |
| `get_ethaddr` / `set_ethaddr` | MAC handling | read/write the station address |

`priv` carries the driver state. There is no `netdev_priv()`; drivers use
`edev->priv` or `container_of()`.

Review checks:

- `send` is **synchronous**. It returns when the frame is out or the timeout
  expired; there is no completion callback and no TX queue in the driver.
- `recv` must call `net_receive(edev, buf, len)` for each frame and must not
  free or reuse the buffer before that call returns. `net_receive()` parses the
  frame inline — ARP and IP handlers run inside it.
- `net_receive()` rejects `len < ETHER_HDR_SIZE` itself, so a driver need not
  check for runts, but it **must** pass the real length, not the descriptor
  size.
- An `open` that does not report link state, or a `recv` that spins waiting for
  a frame, hangs the boot. `recv` is called from the poller and must return
  promptly whether or not a frame arrived.

## Packet buffers

There is no `sk_buff`. A packet is a plain buffer:

```c
#define PKTSIZE 1536
static inline char *net_alloc_packet(void) { return dma_alloc(PKTSIZE); }
```

- `net_alloc_packet()` / `net_free_packet()` for one buffer,
  `net_alloc_packets(void **, count)` / `net_free_packets()` for a ring.
  `net_alloc_packets()` unwinds its own partial allocation on failure.
- Buffers are `dma_alloc()`ed, therefore cache-line aligned and suitable as DMA
  targets. A driver that hands a stack or `malloc()` buffer to DMA is a real
  finding.
- `PKTSIZE` is 1536. A driver programming a larger RX buffer size into the
  hardware than it allocated is an overflow; check the two against each other.

## DMA and cache maintenance

This is where real barebox network driver bugs live, and it is worth the
review time that Linux reviews spend on locking.

- `dma_map_single()` / `dma_unmap_single()`, `dma_sync_single_for_cpu()` /
  `dma_sync_single_for_device()` with `DMA_FROM_DEVICE` / `DMA_TO_DEVICE`.
- RX: sync **for CPU** before reading the descriptor's data, sync **for device**
  after handing the buffer back to the ring. Missing the first read gives stale
  data; missing the second lets the CPU's dirty lines land on top of a DMA
  write.
- TX: sync for device *after* filling the buffer, *before* setting the OWN bit.
  A common port bug is doing it in the other order.
- Sync the actual frame length, not `PKTSIZE`, on TX; sync at least the frame
  length on RX.
- Descriptor rings themselves must be coherent memory (`dma_alloc_coherent()`),
  not `dma_alloc()`.

`drivers/net/designware.c` is a readable reference for the ordering.

## Re-entrancy: slices, not locks

Barebox is single-threaded (see `../deltas.md` §1.5) but it *is* re-entrant
through the poller. The mechanism is `struct slice`, not a lock:

```c
int eth_send(struct eth_device *edev, void *packet, int length)
{
        if (slice_acquired(eth_device_slice(edev)))
                return eth_queue(edev, packet, length);
        ...
        slice_acquire(eth_device_slice(edev));
        ret = eth_send_raw(edev, packet, length);
        slice_release(eth_device_slice(edev));
}
```

- A `send` issued from inside the device's own poll callback is **queued**, not
  recursed into. That is `net/eth.c`, not the driver's problem, and it is not a
  race.
- What *is* a finding: a driver that touches its rings outside
  `send`/`recv`/`halt` — from a separate poller, a timer callback or a command
  — without going through the slice. `slice_depends_on()` expresses a device
  that needs another one quiescent (an MDIO bus shared with the MAC).
- Do not report "missing lock", "unsynchronised access" or "ABBA" here. Ask
  instead which poller contexts can reach the code.

## PHY and MDIO

- `phy_device_connect(edev, bus, addr, adjust_link, flags, interface)` is the
  normal path; `addr` may be -1 to scan. It parses `phy-handle` /
  `phy-mode` from the DT.
- `of_phy_register_fixed_link()` handles `fixed-link` nodes.
- An MDIO bus is a `struct mii_bus` with `read`/`write`/`reset`, registered
  with `mdiobus_register()`. The bus must be registered before the MAC tries to
  connect a PHY on it — with `register_driver()` semantics that means initcall
  level, not file order (`drivers.md`).
- `adjust_link` is where the MAC learns the negotiated speed/duplex. A port
  that drops it, or that programs the MAC once in `open()` instead, works on
  the bench at one link speed and fails on another. Real finding.

## Protocol side

- `net_udp_new(dest, dport, handler, ctx)` / `net_icmp_new()` return a
  `struct net_connection`; release with `net_unregister()`. Not sockets — a
  handler is called from the poll loop with the packet.
- `net_udp_send(con, len)` sends what was written into `con->packet`.
- Header access via `net_eth_to_iphdr()`, `net_eth_to_udphdr()`,
  `net_eth_to_icmphdr()`, `net_eth_to_udp_payload()`.
- Checksums: `net_checksum()` / `net_checksum_ok()`.
- Byte order is explicit — `ntohs()`/`htonl()` and the `__be*` types, same as
  Linux. Reading a length or ethertype without conversion is a real finding.
- `net_route()` picks the egress device; `net_set_ip()`, `net_set_gateway()`,
  `net_set_nameserver()` configure it. There is no routing table to protect
  with RCU.

## Device tree

- MAC address: `of_get_mac_address()` in `drivers/of/of_net.c` already walks
  `mac-address`, `local-mac-address`, `address` and the `mac-address` nvmem
  cell, in that order. A driver inventing its own MAC read is duplicating core
  work; `of_eth_register_ethaddr()` is the registration side.
- Do not confuse `phy-mode` (`phy-connection-type`) handling with Linux'; the
  barebox core parses it in `phy_device_connect()`.
- `barebox,` DT additions for networking are documented under
  `Documentation/devicetree/bindings/` in the barebox tree — see
  `../deltas.md` §1.4 before dismissing a missing-binding comment.

## Quick checks

- `eth_register()` called, `send`/`recv`/`open`/`halt` all set.
- `recv` calls `net_receive()` with the received length and returns promptly.
- Every DMA buffer is `dma_alloc()`ed; every hand-off is bracketed by the
  matching `dma_sync_single_for_{cpu,device}()`.
- RX buffer size programmed into hardware ≤ `PKTSIZE`.
- `adjust_link` applies speed and duplex to the MAC.
- No `sk_buff`, `napi_*`, `netdev_*` or `ethtool_*` in the patch — those are
  signs of a Linux port that has not been converted.
