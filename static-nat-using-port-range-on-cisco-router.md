
---
# Static NAT using Port Range on Cisco Router with existing NAT Overload
---

## Video Walkthroughs
Static NAT using Port Range on Cisco Router with existing NAT Overload - 

## Scenario:

![alt text](https://github.com/hackd-art/networking-tips-and-tricks/blob/main/static-nat-using-port-range-on-cisco-router.png)

1. 3rd Party Public Server (100.100.100.100) should be able to access the server box inside customer network
2. There is an existing NAT overload using the same public IP of the customer router for user traffic going to the internet.
3. Static NAT (Private: 10.11.12.14 - Public: 64.65.66.67) should be implemented but should only allow below ports inbound:

  TCP 5001<br>
  TCP/UDP 5002<br>
  UDP 5003<br>
  UDP 10000-10999<br>

## Existing NAT Overload Configuration

```cisco
inteface GigabitEthernet0/0
 ip address 64.65.66.67 255.255.255.0
 ip nat outside

interface GigabitEthernet0/1
 ip address 10.11.12.13 255.255.255.0
 ip nat inside

ip nat inside source list NAT-OVERLOAD interface GigabitEthernet0/0 overload

ip access-list extended NAT-OVERLOAD
 deny ip 10.11.12.0 0.0.0.255 10.0.0.0 0.255.255.255
 deny ip 10.11.12.0 0.0.0.255 172.16.0.0 0.15.255.255
 deny ip 10.11.12.0 0.0.0.255 192.168.0.0 0.0.255.255
 permit ip 10.11.12.0 0.0.0.255 any
```

## Static NAT using Port Range Configuration

```cisco

ip nat inside source static 10.11.12.14 64.65.66.67 route-map RM-PORT-RANGE reversible extendable

ip access-list extended ACL-PORT-RANGE
 permit tcp host 10.11.12.14 range 5001 5002 host 100.100.100.100
 permit udp host 10.11.12.14 range 5002 5003 host 100.100.100.100
 permit udp host 10.11.12.14 range 10000 10999 host 100.100.100.100
 permit tcp host 100.100.100.100 host 10.11.12.14 range 5001 5002
 permit udp host 100.100.100.100 host 10.11.12.14 range 5002 5003
 permit udp host 100.100.100.100 host 10.11.12.14 range 10000 10999

route-map RM-PORT-RANGE
 match ip address ACL-PORT-RANGE
```

