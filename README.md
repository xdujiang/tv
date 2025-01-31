# Note about IPTV

1. Prepare
   1. Capture package
      1. Target
         1. Get IPoE auth parameters
         2. Get channel list
      2. Method
         1. Port mirror on switch or router
         2. tcpdump on ONU
   2. IPoE parameters
      - Filter "DHCP" protocol in wireshark, check the "DHCP Discover" frame, get the Option 12, Option 55, Option 60.
   3. Channel list generation
      1. Extract from captured package
         1. In wireshark, File → Export Objects → HTTP...
         2. Find "frameset_builder.jsp" in list, save the larger one
         3. Get channel list in file
      2. Scanning and validation
          - No authorization needed to receive rtp data in currect area. It works even without correct IP/GW.
         1. Plug network cable into IPTV port
         2. Disable all unused NIC
         3. Get privilaged cmd, run iptvscanner.exe to get channel list
         4. Import channel list to 直源检测验证管理工具, remove invalid channel
2. OpenWRT configuration
    1. Switch setting
       1. Network enviroment
          1. ONU: no VLAN tag for internet data, VLAN 3000 for IPTV data. IPTV bonded to lan1 port.
          2. Lan1 connected to WAN port of the router in living room. Router get IPTV data untagged and bonded to lan1.
          3. Add OpenWRT router between ONU and router.
          4. Additional Requirement:
              - Dual WAN port for OpenWRT, all WAN port work as switch.
              - All internet and IPTV data passed through to downstream router or switch.
              - Keep network topology similiar to original, and no setting required to remove the OpenWRT router.
       2. Setting dual WAN
          1. Essential knowledge:
             - Untagged data will be VLAN tagged with PVID (port VLAN ID) while ingressing to switch (refers to internet data in current topology).
             - Tagged data will keep original tag if allowed to ingress.
             - OpenWRT won't deal with VLAN on a network bridge with VLAN filter disabled.
             - Once tried to extract specific VLAN data from a network bridge (like setting device `wan.3000`), all VLAN should be corrected set.
             - PVID will be the primary VLAN ID if set.
             - In following settings, VLAN 100 is set as primary VLAN of eth0, so all untagged data ingressed to eth0 will be tagged with VLAN ID 100.
             - ![VLAN setting](image-1.png)
       3. Setting VLAN
          1. Remove an eth port from br-lan, e.g. eth3 as wan2 port
          2. Add a bridge device `wan-bridge` connecting wan port (eth0) and eth3
          3. Enable VLAN filtering. Add VLAN 100 as internet data, and 3000 as IPTV data.
          4. Set VLAN 100 untagged and primary to wan port, meanwhile untagged to eth3, so downstream router can still get untagged internet data.
              - Internet data flows untagged between ONU and OpenWRT, as well as between OpenWRT and downstream router.
          5. Set VLAN 3000 tagged to wan port, meanwhile tagged to eth3, so downstream router can still extract IPTV data with VLAN 3000.
              - Internet data flows tagged with VLAN ID 3000 between ONU and OpenWRT, as well as between OpenWRT and downstream router.
          6. Change device to `wan-bridge.100` in wan interface setting.
       4. IPoE setting
          1. Add new interface `iptv` with device `wan-bridge.3000`, choose DHCP protocol.
          2. Set all DHCP options [https://v2ex.com/t/896669]
              - If Option 60 is binary data, file `/lib/netifd/proto/dhcp.sh` should be modified. Replace line `${vendorid:+-V "$vendorid"}`  with `${vendorid:+-V "" "-x 0x3c:$vendorid"}`
          3. OpenWRT may send two Option 55 in the DHCP Discover, but it works.
          4. Check IP address.
    2. IGMP proxy and udpxy
       1. Install igmpproxy or omcproxy, choose `iptv` as uplink interface, `lan` as downlink interface.
       2. Install udpxy or msd_lite, bind to `br-lan`, and set `wan-bridge.3000` as source.
       3. Add traffic rules in firewall setting.
          1. Allow IGMP from iptv to `this device`
          2. Allow UDP from iptv to lan and `this device`, IP `224.0.0.0/4`
3. IPTV player
4. EPG
   1. http://epg.51zmt.top:8000/
   2. https://epg.112114.xyz/
