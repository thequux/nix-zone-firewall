Zone-based firewall for NixOS
=============================

This is mostly undocumented, incomplete, and subject to change
(possibly massively), but for anybody whose bravery is only exceded by
their foolhardiness, have fun!

To get you started, here's the relevant snippet from my configuration.nix:

```nix
{
  imports = [
    ./nix-zone-firewall
  ];
  
  # The zone-based portion currently doesn't support the nat hooks,
  # so we define a table for them
  # 
  # If you'd like to insert your own chains and rules into the 
  # zone-based firewall generator, that all ends up in 
  #   networking.nftables.tables.inet.nixos-zfw
  networking.nftables.tables.ip.nat =
    let
      if_wan = "enp1s0";
      
    in {
      chains.prerouting = {
        hook = {
          type = "nat";
          hook = "prerouting";
          priority = "dstnat";
        };

        rules = [
          {
            rule = [
              "iifname ${if_wan} tcp dport 4901 dnat to 10.24.74.32:22"
              "iifname ${if_wan} udp dport 5100 dnat to 10.24.74.32:5100"
              "iifname ${if_wan} tcp dport {80,443} dnat to 10.24.74.18"
            ];
          }
        ];
      };

      chains.postrouting = {
        hook = {
          type = "nat";
          hook = "postrouting";
          priority = "srcnat";
        };

        rules = [
          { rule = "counter meta oifname ${if_wan} masquerade"; }
        ];
      };

      chains.pkt-trace = {};
    };
  networking.zone-firewall =
    let
      vpns = [ "vpn_oshaberi" ];
      private = [ "lan" "mgmt"  "vpn_oshaberi"];
    in {
      interfaces = {
        enp1s0 = "wan";
        br10 = "lan";
        br1 = "mgmt";
        tap0 = "lan";
        ztkytnifbj = "lan";
        vpn-oshaberi = "vpn_oshaberi";
      };
      
      rules = [
        {
          fromZone = "lan";
          toZone = "local";
          rule = "tcp dport 22 accept";
          priority = 1000;
        }
        {
          toZone = "local";
          rule = [
            "tcp dport 22 accept"
            # VPNs
            "udp dport { 1194, 1195 } accept"
          ];
        }
        
        {
          # Allow OSPF from VPNs
          fromZone = vpns ++ ["lan"];
          toZone = "local";
          rule = [
            "ip protocol 89 accept"
            "tcp dport 179 accept"
          ];
        }
        {
          fromZone = "local";
          toZone = vpns ++ ["lan"];
          rule = [
            "ip protocol 89 accept"
            "tcp dport 179 accept"
          ];
        }
        {
          fromZone = "wan";
          toZone = "lan";
          rule = [
            "ip daddr 10.24.74.32 tcp dport 22 accept"
            "ip daddr 10.24.74.18 tcp dport { 80, 443 } accept"

            # Elite dangerous
            "ip daddr 10.24.74.32 udp dport 5100 accept"
          ];
        }

        {
          fromZone = "local";
          rule = "accept";
          priority = -300;
        }

        {
          fromZone = [ "mgmt" "lan" ];
          toZone = "local";
          rule = "accept";
          priority = -300;
        }

        {
          fromZone = [ "lan" ];
          toZone = [ "vpn_oshaberi" "wan" ];
          rule = "accept";
          priority = -300;
        }
        
        { fromZone = ["mgmt"]; rule = "drop"; priority = -500; }
        { toZone = ["mgmt"]; rule = "drop"; priority = -500; }
      ];
      enable = true;
    };
}
```
