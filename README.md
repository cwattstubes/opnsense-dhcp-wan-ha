# opnsense-dhcp-wan-ha
A set of scripts to have high availability DHCP WAN utilizing CARP Events on OpnSense

OPNsense single-DHCP-WAN HA: migration kit for RT1 / RT2
============================================================

This kit is tailored for:
  WAN = vtnet0 / friendly name wan / IPv4 DHCP
  LAN CARP = 1@vtnet1 / VIP 192.168.x.1
  RT1 LAN = 192.168.x.2, RT2 LAN = 192.168.x.3

Files:
  wan-ha-control   -> /usr/local/sbin/wan-ha-control
  50-wan-ha        -> /usr/local/etc/rc.syshook.d/carp/50-wan-ha
  90-wan-ha-start  -> /usr/local/etc/rc.syshook.d/start/90-wan-ha-start

Important: these are custom code, not vendor-supported HA automation. The shell
scripts passed syntax checks, and embedded PHP passed php -l in a test runtime,
but the OPNsense runtime behavior has NOT been live-tested. Use console access,
configuration backups and a maintenance window. This is not a hard fencing
solution: WAN overlap is possible during a transition or early boot.

1. While RT1 remains CARP MASTER and WAN-active, back up config on EACH node:
   cp -p /conf/config.xml /root/config-before-wan-ha.xml


2. Transfer these 3 files to /tmp/ on EACH node. Install only the controller
   first, with no active CARP hook:
      install -m 755 /tmp/wan-ha-control /usr/local/sbin/wan-ha-control
      sh -n /usr/local/sbin/wan-ha-control
      /usr/local/sbin/wan-ha-control status

3. On BACKUP RT2, explicitly reconcile while RT1 is still MASTER:
      /usr/local/sbin/wan-ha-control apply
   Verify no IPv4 address / no DHCP client on vtnet0.
   On MASTER RT1, reconcile:
      /usr/local/sbin/wan-ha-control apply
   Verify WAN IPv4, DHCP and default route. Logs: /var/log/wan-ha.log.

4. Install CARP hooks on both nodes:
      install -m 755 /tmp/50-wan-ha /usr/local/etc/rc.syshook.d/carp/50-wan-ha
      sh -n /usr/local/etc/rc.syshook.d/carp/50-wan-ha

5. In a maintenance window put RT1 in Persistent CARP Maintenance Mode.
   Verify RT2 is MASTER, WAN DHCP has an address, default route exists,
   and /var/log/wan-ha.log shows completion. Watch DHCP capture if not.
   Restore RT1 and check RT2 becomes BACKUP, relinquishes WAN, and RT1
   reacquires DHCP. Confirm no simultaneous lease on both nodes.

6. AFTER live transition tests pass, install the post-boot reconciliation hook:
      mkdir -p /usr/local/etc/rc.syshook.d/start
      install -m 755 /tmp/90-wan-ha-start /usr/local/etc/rc.syshook.d/start/90-wan-ha-start
   This runs after networking starts; it is not pre-boot fencing.

Rollback: restore original /conf/config.xml from the saved backup using the
OPNsense GUI/console restore workflow; move all new hooks out of their hook
folders, remove the new controller, restore archived scripts only if you intend
to return to that older HA implementation. Restore original WAN enablement on
both nodes deliberately; avoid running two controllers at once.
