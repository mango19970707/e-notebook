### 使用scap构建镜像流量

---

```py
import pytest
import requests
import time
from scapy.all import *
 
 
@pytest.mark.usefixtures("get_config_instance")
def test_76_vlan(get_config_instance):
 
    # Define ethernet and IP/TCP layers
    eth = Ether()
    ip = IP(src='10.0.0.2', dst='10.0.0.1')
 
    # TCP header
    tcp = TCP(sport=8080, dport=80)
 
 
    # Define VLAN tags
    outer_vlan = Dot1Q(vlan=10)
 
    # Assemble request
    http_request = "GET /vlan HTTP/1.1\r\nHost: example.com\r\n\r\n"
    pkt = eth / outer_vlan / ip / tcp / http_request
    sendp(pkt, iface = get_config_instance.get_traffic_interface())
 
 
    # Ethernet header
    eth = Ether()
 
    # Outer VLAN tag
    outer_vlan = Dot1Q(vlan=100)
 
    # Inner VLAN tag
 
    # IP header
    ip = IP(src='10.0.0.1', dst='10.0.0.2')
 
    # TCP header
    tcp = TCP(sport=80, dport=8080)
 
    # HTTP response
    http = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n<html>Hello World!</html>"
 
    # Assemble packet
    pkt = eth / outer_vlan / ip / tcp / Raw(load=http)
 
    # Show packet
    pkt.show()
 
    # Send packet
    sendp(pkt, iface= get_config_instance.get_traffic_interface())
 
    time.sleep(5)
 
    url = get_config_instance.get_clickhouse_addr()
    data = '''SELECT method FROM access
              WHERE ts > now()-20
              AND host = 'example.com'
              AND ip = '10.0.0.2'
              AND url = '/vlan'
              LIMIT 1;'''.encode('utf-8')
    check = requests.post(url, data=data, verify=False)
 
    assert "GET" in check.text
```