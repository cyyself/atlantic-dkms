# atlantic driver with page_pool RX — DKMS package

This package carries the Marvell/Aquantia AQtion (atlantic) driver from
Linux 7.2-rc2 with the RX buffer management converted to the kernel's
generic page_pool API, so it can be built and run on the currently
installed distribution kernel via DKMS (or a plain out-of-tree make).

## Motivation

On systems where the NIC sits behind an IOMMU (e.g. AMD Strix Halo with a
Thunderbolt-attached QNAP QNA-T310G1S 10G adapter), the stock driver
cannot reach line rate: every RX buffer was allocated with
`dev_alloc_pages()` and mapped with `dma_map_page()`, and unmapped/freed
once the stack consumed the packet.  With an IOMMU, every map/unmap is an
IOTLB/pagetable operation, and at 10G packet rates this dominates the RX
path (~2.2 Gbit/s ceiling with MTU 1500 TCP).

An [earlier workaround](https://lore.kernel.org/lkml/tencent_E71C2F71D9631843941A5DF87204D1B5B509@qq.com/) made the RX page order tunable via a module
parameter (`rxpageorder=3`), amortizing one map/unmap over 8 pages worth
of frames.  Converting to page_pool is the proper upstream fix: pages are
DMA-mapped once when they enter the pool and stay mapped while they are
recycled between the driver and the network stack, so steady-state RX
does no IOMMU work at all.

## Implementation details

The conversion replaces the driver's private RX page lifecycle
(`aq_rxpage`: `dev_alloc_pages()` + `dma_map_page()` + a "page flip"
scheme that subdivided high-order pages by hand and reused them based on
`page_ref_count()`) with a page_pool instance per RX ring:

* `aq_ring_rx_alloc()` creates a page_pool (`page_pool_create()`) with
  `PP_FLAG_DMA_MAP | PP_FLAG_DMA_SYNC_DEV`, order = the ring's page order
  (order 0 normally, `AQ_CFG_XDP_PAGEORDER` = 2 when an XDP program is
  attached, or `cfg->rxpageorder` if larger), pool_size = the number of
  RX descriptors, dma_dir = `DMA_FROM_DEVICE` and max_len covering the
  whole page.  The pool is destroyed in `aq_ring_free()` (for the vector
  rings this runs after `xdp_rxq_info_unreg()`, as required).

* `aq_get_rxpages()` now takes one page fragment per RX descriptor from
  the pool via `page_pool_dev_alloc_frag()`.  The fragment size is
  page_offset (XDP headroom) + frame_max (2 KiB) + tail_size (XDP
  tailroom), so the pool's fragment allocator does the sub-page
  splitting the old flip scheme did by hand, including for high-order
  pages.  `rxdata.daddr` is the pool-managed mapping
  (`page_pool_get_dma_addr()`); descriptors are still programmed with
  daddr + pg_off as before.

* Buffer ownership is now transfer-based instead of refcount-based:

  - Normal RX (`__aq_ring_rx_clean()`): the header is copied into a
    `napi_alloc_skb()` head as before; if payload remains, the page is
    attached with `skb_add_rx_frag()` and the ring drops its reference
    by clearing `rxdata.page` (the old code did `page_ref_inc()` and
    kept the page).  Multi-descriptor (RSC/jumbo) chains transfer each
    buffer's page the same way.  skbs are marked with
    `skb_mark_for_recycle()`, so when the stack frees them the pages
    return to the pool (still mapped) instead of going back to the page
    allocator.

  - XDP (`__aq_ring_xdp_clean()`): the xdp_rxq memory model is now
    `MEM_TYPE_PAGE_POOL` (registered per ring in `aq_vec_ring_alloc()`,
    which now allocates the ring before registering the rxq info so the
    pool exists).  The `xdp_buff` owns the fragments as soon as they are
    attached, and every verdict returns them through the memory model:
    `xdp_return_buff()` on XDP_DROP/ABORTED, TX-completion
    `xdp_return_frame()` for XDP_TX (plain, not `_rx_napi`, since
    `ndo_xdp_xmit` frames may belong to a pool owned by another NAPI
    context), and `__xdp_build_skb_from_frame()` marks page_pool-backed
    skbs for recycling on XDP_PASS.

  - Packets that never leave the driver (HW error descriptors, RSC
    aborts, header-only frames whose payload was fully copied) keep
    their page in the ring buffer: `aq_get_rxpages()` sees
    `rxdata.page != NULL` and reposts the same fragment to the
    hardware, which is cheaper than a pool round-trip.

  - `aq_ring_rx_deinit()` walks the entire ring (consumed but not yet
    refilled slots can sit outside `[sw_head, sw_tail)`) and returns any
    fragment still held with `page_pool_put_full_page()`; missing any
    would stall `page_pool_destroy()` forever.

* The PTP RX ring uses the same `aq_ring_rx_alloc()`/`aq_ring_rx_clean()`
  code and therefore gets its own small pool; it now registers its
  xdp_rxq with the `MEM_TYPE_PAGE_POOL` memory model too (previously its
  xdp_rxq was never registered, which after the conversion would have
  freed pool pages through the wrong return path once any XDP program
  was attached).  The hardware-timestamp ring (hwts_rx) carries no page
  buffers and is unaffected.

* The PageFlips/PageReuses/PageFrees ethtool per-queue counters counted
  events of the removed flip scheme and are gone; AllocFails now counts
  pool fragment allocation failures.  `CONFIG_AQTION` now selects
  `CONFIG_PAGE_POOL`.

## Performance

`iperf3 -R` (peer sending) over IPv6 link-local, MTU 1500, QNAP
QNA-T310G1S (AQC100 family) behind Thunderbolt/IOMMU:

| driver                                            | throughput  |
|----------------------------------------------------|-------------|
| stock driver (order-0 pages, map/unmap per page)   | 2.24 Gbit/s |
| page_pool driver (this package)                    | 9.14 Gbit/s |

Also verified on the live system: ifdown/ifup cycles and module
unload/reload complete without "stalled pool shutdown" warnings (no
leaked pool references), and the `ethtool -S` per-queue layout is
consistent after the removal of the three page-flip counters.

## Building

On this machine the package is already registered and installed:
the source is copied to `/usr/src/atlantic-7.2-rc2-pagepool`, `dkms
status` reports `atlantic/7.2-rc2-pagepool` as installed, and the module
lives in `/lib/modules/$(uname -r)/updates/dkms/atlantic.ko.xz`,
overriding the stock driver from the next boot onwards (the running
kernel already has it loaded).

To set it up elsewhere with DKMS:

```sh
sudo cp -r /home/cyy/atlantic-dkms /usr/src/atlantic-7.2-rc2-pagepool
sudo dkms add atlantic/7.2-rc2-pagepool
sudo dkms install atlantic/7.2-rc2-pagepool
echo atlantic | sudo tee -a /etc/initramfs-tools/modules
sudo update-initramfs -u
```

DKMS rebuilds the module automatically for every newly installed kernel
(`AUTOINSTALL=yes`).  The module installs into `/updates/dkms`, which
overrides the in-kernel atlantic.ko.

The initramfs steps matter on Debian: with `MODULES=most` the initramfs
includes network drivers, and it is generated *before* the DKMS install,
so the first reboot would otherwise load the stock module from the
initramfs during early boot (DKMS only refreshes the initramfs when a
new kernel is installed, not when a module is added to the running
one).  Listing `atlantic` in `/etc/initramfs-tools/modules` also
guarantees the module is picked up via depmod (which resolves to
`updates/dkms`) rather than the directory scan, which no longer finds
it after dkms archives the stock `.ko.xz`.

Without DKMS:

```sh
cd src
make -C /lib/modules/$(uname -r)/build M=$PWD CONFIG_AQTION=m modules
sudo rmmod atlantic
sudo insmod ./atlantic.ko
```

Reverting: `sudo dkms remove atlantic/7.2-rc2-pagepool --all` (or rmmod
the module) and reload the distribution module with `sudo modprobe
atlantic`.

## Source

`src/` is a verbatim copy of `drivers/net/ethernet/aquantia/atlantic/`
from the kernel tree at `/home/cyy/linux`, branch `atlantic_pagepool`,
commit "net: atlantic: convert RX path to page_pool".  The same change
is intended for upstream submission to netdev; this package only exists
so the running distribution kernel can use it before it lands.

## License

GPL-2.0-only.  The driver source in `src/` is copied from the Linux
kernel and every file carries an `SPDX-License-Identifier: GPL-2.0-only`
tag; the packaging files (dkms.conf, this README) are under the same
license.  The full license text is in [COPYING](COPYING).
