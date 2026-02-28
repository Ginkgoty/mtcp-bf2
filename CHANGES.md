# Change Summary on BlueField-2 DPU

## Envrionment
- Nvidia DOCA 1.5.5 LTS  
- MLNX_DPDK 20.11

You may need install necessary tools firstly:

```bash
sudo apt-get install -y autoconf automake libtool
```

## Modified Files (7)

| File            | Changes                                                                                                                                                                                                                                                                     |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Makefile.in`   | Migrated the build system from `include $(RTE_SDK)/mk/rte.vars.mk` to using `pkg-config --cflags/--libs libdpdk`, while retaining the legacy fallback path.                                                                                                                 |
| `Makefile.in`   | Switched DPDK linking from `ldflags.txt` to `pkg-config --libs libdpdk`.                                                                                                                                                                                                    |
| `Makefile.in`   | Same as above.                                                                                                                                                                                                                                                              |
| `configure.ac`  | Updated the comment for `--with-dpdk-lib` (removed the hard-coded 17.08 version reference).                                                                                                                                                                                 |
| `dpdk_module.c` | Five fixes: replaced `RTE_MBUF_PREFETCH_TO_FREE` with `rte_prefetch0`; added version guards for `struct ether_hdr / ether_addr / ipv4_hdr`; added guards for `sizeof(struct ether_hdr)` in the LRO section; added pre-22.11 forward compatibility for `PKT_RX_*_CKSUM_BAD`. |
| `io_module.c`   | Made `probe_all_rte_devices()` fault-tolerant: if `/dev/dpdk-iface` is missing, do not exit; instead allow EAL to auto-probe (mlx5 bifurcated mode). Replaced `-w` with `-a`. Replaced `eal_proc_type_detect()` with `rte_eal_process_type()`.                              |
| `core.c`        | Replaced `rte_get_master_lcore()` with `rte_get_main_lcore()`. Removed direct access to `lcore_config[]` (not a public API since DPDK 20.11). Added `#include <rte_version.h>`.                                                                                             |
| `config.c`      | Fixed the `strncpy` truncation warning.                                                                                                                                                                                                                                     |

## Newly Added Configuration Files

| File               | Purpose                                                              |
| ------------------ | -------------------------------------------------------------------- |
| `epserver-sf.conf` | mTCP configuration template (`port = enp3s0f0s0`, `num_mem_ch = 1`). |
| `arp.conf`         | Static ARP table template.                                           |
| `route.conf`       | Static routing table template.                                       |

## Usage

0. **Set Hugepages**: 
1. **Assign an IP to the SF interface**: `sudo ip addr add <IP>/<MASK> dev enp3s0f0s0`
2. **Edit configuration files**: fill in the peer information in `arp.conf` and `route.conf`
3. **Run**: `cd apps/example && sudo ./epserver -f epserver-sf.conf -p www 2>&1`

**Key Notes**:

* **No need** to run `setup_mtcp_dpdk_env.sh` (this script only applies to Intel NICs + source-built DPDK).
* **No need** to build `dpdk-iface-kmod` (the mlx5 bifurcated driver already provides the `enp3s0f0s0` interface in the kernel).
* To rebuild, set
  `export PKG_CONFIG_PATH=/opt/mellanox/dpdk/lib/aarch64-linux-gnu/pkgconfig:$PKG_CONFIG_PATH`
  then run: `./configure --with-dpdk-lib=/opt/mellanox/dpdk && make`
