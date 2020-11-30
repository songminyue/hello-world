# Networking-afc Plugin for OpenStack Neutron

## Overview
Based on the OpenStack Networking ML2 plugin which provides an extensible architecture that supports multiple independent drivers to be used to configure different network devices from different vendors, Networking-afc Neutron Plugin offers infrastructure services for Asterfusion’s  switches and configure the underlying physical network with cooperation of Asteris Fabric SDN Controller (AFC for short). Networking-afc Neutron Plugin can efficiently offload Layer 2 and layer 3 network functions on switches to allow VM traffic for North-South & East-West while to improve the performance of compute nodes in the cloud environment.

Note. As a unified visual platform of SDN controller, AFC docks over business requirements from the overlay network and frees up the capability of the Asterfusion basic network.

### Architectural
Networking-afc Neutron Plugin 

![NETWORKING-AFC.png](https://github.com/songminyue/hello-world/NETWORKING-AFC.png)

### Supported components:

|components||
|:-----------------:|:---------------------------:|
|Openstack version   |Rocky                        |
|Asterfusion switches|CX306P|
|Linux Distribution  |Centos7.6|
|Type driver         |Aster_vxlan, Aster_ext_net|
|Mechanism Driver    |AsterCXSwitchMechanismDriver|

## Prerequsites

* Openstack Rocky is required
* Ensure that lldp is installed and started on all compute nodes
* Ensure that pip is enabled on controller

## Configuration

1.  ML2 plugin Configuration on the controller
> add type_driver and mechanism_drivers of asterfusion in /etc/neutron/plugins/ml2/ml2_conf.ini
```    
[ml2]
    mechanism_drivers=asterswitch,openvswitch
    type_drivers=vxlan,flat,vlan,local,aster_vxlan, aster_ext_net
    tenant_network_types=aster_vxlan
```
  physnet_4_102/physnet_4_105 is the provider number of asterfusion’s switch, following “30:50” is the vlan range which can be same within two switches but have to be in the range 2-4094
```
[ml2_type_vlan]
    network_vlan_ranges=physnet_4_102:30:50,physnet_4_105:100:200
    
[ml2_type_aster_vxlan]
    vni_ranges=10000:10010
    l3_vni_ranges=10:20
    mcast_ranges=224.0.0.1:224.0.0.3
    # Border leaf l2 vni ranages
    l2_vni_ranges=100:110
    
[ml2_type_aster_ext_net]
    aster_ext_net_networks=fw1,fw2
```
  External network configuration where physical_network_ports_mapping shows the border’s interface connected with cooresponding External network
```
[ml2_border_leaf:192.168.4.102]
    # Border leaf connect FW vlan ranges and interface_names
    vlan_ranges=30:50
    physical_network_ports_mapping=fw1:[X28],fw2:[X27, X29]
```
  host_ports_mapping is the maping of node’s hostname and interfaces of cx connected with node
```
[ml2_mech_aster_cx:192.168.4.102]
    physnet=physnet_4_102
    host_ports_mapping=controller:[X25],computer1:[X29],computer2:[X37]
    
[ml2_mech_aster_cx:192.168.4.105]
    physnet=physnet_4_105
    host_ports_mapping=controller:[X25],computer1:[X29],computer2:[X37]
```
  configurate the parameters of AFC
```
[aster_authtoken]
    username=aster
    password=123456
    auth_uri=http://192.168.4.169:9696
    is_send_afc=True
```
> add plugin configuration in /etc/neutron/neutron.conf
```
service_plugins=afc_l3
```

2.  add supported_provider_types “aster_vxlan”, “aster_ext_net” in /usr/share/openstack-dashboard/openstack_dashboard/local/local_settings.py on the node where horizon is deployed
```
OPENSTACK_NEUTRON_NETWORK = {
  …… ……
  'supported_provider_types': ['local', 'flat', 'vlan', 'gre', 'vxlan', 'geneve', "aster_vxlan", "aster_ext_net"],
        'extra_provider_types': {
            "aster_ext_net": {
              'display_name': _('AsterExtNet'),
              'require_physical_network': True,
              'require_segmentation_id': False
            },
            "aster_vxlan": {
              'display_name': _('AsterVXLAN'),
               'require_physical_network': False,
               'require_segmentation_id': True,
            },
             …… ……
         },

  'segmentation_id_range': {
            'vlan': (1, 4094),
            'gre': (1, (2 ** 32) - 1),
            'vxlan': (1, (2 ** 24) - 1),
            'geneve': (1, (2 ** 24) - 1),
            'aster_vxlan': (1, (2 ** 24) - 1),
        }
}
        
        
```

3.  All nodes with neutron-openvswitch-agent installed need to be configured in /etc/neutron/plugins/ml2/openvswitch_agent.ini, firstly comment out the tunnel_types option, then add bridge_mappings and restart neutron-openvswitch-agent
```
bridge_mappings=physnet_4_102:br_4_102
systemctl restart neutron-openvswitch-agent
```
4.  Install
install networking-afc plugin and update database
```
pip install networking_afc_v1.0-xxxxx.whl (network link)
neutron-db-manage --subproject networking_afc upgrade head
```

5.  Restart service on controller
```
systemctl restart httpd 
systemctl restart neutron-server
```
