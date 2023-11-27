# LinuxBridge OpenStack install


Relatively simple DevStack configuration with Linux Bridge

> Still does not work.
>
> There is no reported error, but VM creation fails with ERROR and 
> `openstack network agent list` is empty which is problem....


Requirements
* Ubuntu 22.04 LTS
* main network card configure with static IP address (using `eth0`)

# Setup

You need Ubuntu 22.04 LTS with properly configured static IP address.
Both `hostname` and `hostname -f` must properly
resolve to that static IP address of host.

Prepare DevStack usual way:
```shell
sudo apt-get install git-core
mkdir -p ~/projects
cd ~/projects
# Branch 2023.1 is known as "Antelope" release
git clone -b stable/2023.1 https://opendev.org/openstack/devstack.git
git clone https://github.com/hpaluch/devstack-examples.git
cd ~/projects/devstack
cp ../devstack-examples/macvtap/local.conf local.conf
```

Finally double-check `~/projects/devstack/local.conf` for correctness especially
regarding all IP addresses and then run:

```shell
./stack.sh
```

At the end you should see summary like:
```
This is your host IP address: 192.168.0.8
This is your host IPv6 address: 2a02:d8a0:a:1c29:e3:a0ff:fe4e:339c
Horizon is now available at http://192.168.0.8/dashboard
Keystone is serving at http://192.168.0.8/identity/
The default users are: admin and demo
The password: Secret123

Services are running under systemd unit files.
For more information see: 
https://docs.openstack.org/devstack/latest/systemd.html

DevStack Version: 2023.1
Change: 146bf0d10c82e1705eb1c01e0f0d4cdf410c788c Merge "git: git checkout for a commit hash combinated with depth argument" into stable/2023.1 2023-08-12 12:27:05 +0000
OS Version: Ubuntu 22.04 jammy
```

To start first VM you should verify that all necessary items are configured properly:

```shell
# start new shell to not cluter main environment
bash
source ~/projects/devstack/openrc admin
PS1="$OS_USERNAME@openstack $PS1"
# run main openstack shell to speed up queries
# Plase note that openstack commadn response is much longer for admin
openstack
```
Let's query network(s) first:
```
(openstack) network list
+--------------------------------------+---------+--------------------------------------+
| ID                                   | Name    | Subnets                              |
+--------------------------------------+---------+--------------------------------------+
| 3812fa5a-6fc8-4fef-87a6-b3717e7fcc1f | private | 7bfa1400-f2fa-48bd-b6ec-f861cf939b86 |
| b6a0d2fc-14c0-403e-9a53-665a584df376 | public  | 6125b95d-5995-43d3-9a03-52efb43c949a |
+--------------------------------------+---------+--------------------------------------+

(openstack) subnet list
+--------------------------------------+----------------+--------------------------------------+----------------+
| ID                                   | Name           | Network                              | Subnet         |
+--------------------------------------+----------------+--------------------------------------+----------------+
| 6125b95d-5995-43d3-9a03-52efb43c949a | public-subnet  | b6a0d2fc-14c0-403e-9a53-665a584df376 | 192.168.0.0/24 |
| 7bfa1400-f2fa-48bd-b6ec-f861cf939b86 | private-subnet | 3812fa5a-6fc8-4fef-87a6-b3717e7fcc1f | 10.0.0.0/26    |
+--------------------------------------+----------------+--------------------------------------+----------------+


```
My interest is network `public` with subnet `public-subnet` that corresponds with my real Host network
(192.168.0.8/24)

Now items that are required to start VM:
```
# We need at least one image

(openstack) image list
+--------------------------------------+--------------------------+--------+
| ID                                   | Name                     | Status |
+--------------------------------------+--------------------------+--------+
| 5524b046-c6db-49a5-825f-d12eab48b113 | cirros-0.5.2-x86_64-disk | active |
+--------------------------------------+--------------------------+--------+

# We need at least one flavor

(openstack) flavor list
+----+-----------+-------+------+-----------+-------+-----------+
| ID | Name      |   RAM | Disk | Ephemeral | VCPUs | Is Public |
+----+-----------+-------+------+-----------+-------+-----------+
| 1  | m1.tiny   |   512 |    1 |         0 |     1 | True      |
| 2  | m1.small  |  2048 |   20 |         0 |     1 | True      |
| 3  | m1.medium |  4096 |   40 |         0 |     2 | True      |
| 4  | m1.large  |  8192 |   80 |         0 |     4 | True      |
| 5  | m1.xlarge | 16384 |  160 |         0 |     8 | True      |
| c1 | cirros256 |   256 |    1 |         0 |     1 | True      |
| d1 | ds512M    |   512 |    5 |         0 |     1 | True      |
| d2 | ds1G      |  1024 |   10 |         0 |     1 | True      |
| d3 | ds2G      |  2048 |   10 |         0 |     2 | True      |
| d4 | ds4G      |  4096 |   20 |         0 |     4 | True      |
+----+-----------+-------+------+-----------+-------+-----------+


# We need at least one hypervisor in state "up"

(openstack) hypervisor list
+--------------------------------------+---------------------+-----------------+-------------+-------+
| ID                                   | Hypervisor Hostname | Hypervisor Type | Host IP     | State |
+--------------------------------------+---------------------+-----------------+-------------+-------+
| c360ac19-337d-49a5-96cf-eda1a6d5e51b | devstack3           | QEMU            | 192.168.0.8 | up    |
+--------------------------------------+---------------------+-----------------+-------------+-------+
```

Double check our subnet "public-subnet":

```
(openstack) subnet show public-subnet
+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| allocation_pools     | 192.168.0.128-192.168.0.250          |
| cidr                 | 192.168.0.0/24                       |
| created_at           | 2023-11-27T17:48:24Z                 |
| description          |                                      |
| dns_nameservers      |                                      |
| dns_publish_fixed_ip | None                                 |
| enable_dhcp          | False                                |
| gateway_ip           | 192.168.0.1                          |
| host_routes          |                                      |
| id                   | 6125b95d-5995-43d3-9a03-52efb43c949a |
| ip_version           | 4                                    |
| ipv6_address_mode    | None                                 |
| ipv6_ra_mode         | None                                 |
| name                 | public-subnet                        |
| network_id           | b6a0d2fc-14c0-403e-9a53-665a584df376 |
| project_id           | b532d459bf61461cbac51388a69c22f4     |
| revision_number      | 0                                    |
| segment_id           | None                                 |
| service_types        |                                      |
| subnetpool_id        | None                                 |
| tags                 |                                      |
| updated_at           | 2023-11-27T17:48:24Z                 |
+----------------------+--------------------------------------+
```

And parent Network "public"
```
(openstack) network show public
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2023-11-27T17:48:10Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | b6a0d2fc-14c0-403e-9a53-665a584df376 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | True                                 |
| is_vlan_transparent       | None                                 |
| mtu                       | 1500                                 |
| name                      | public                               |
| port_security_enabled     | True                                 |
| project_id                | b532d459bf61461cbac51388a69c22f4     |
| provider:network_type     | flat                                 |
| provider:physical_network | default                              |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 2                                    |
| router:external           | External                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   | 6125b95d-5995-43d3-9a03-52efb43c949a |
| tags                      |                                      |
| tenant_id                 | b532d459bf61461cbac51388a69c22f4     |
| updated_at                | 2023-11-27T17:48:24Z                 |
+---------------------------+--------------------------------------+
```


Now we can use standard way to define keypair:

```shell
(openstack) exit # exit openstack command shell (but stay in bash)

openstack keypair create adminkp > ~/id_rsa_admin
chmod 600 ~/id_rsa_admin
file ~/id_rsa_admin
```

Now moment of truth - try to create first VM:
```shell
openstack server create --network public --flavor m1.tiny \
     --image cirros-0.5.2-x86_64-disk  --key-name adminkp vm1
```
Then poll this command, but Oooopssssss...
```shell
openstack server list

+--------------------------------------+------+--------+----------+--------------------------+---------+
| ID                                   | Name | Status | Networks | Image                    | Flavor  |
+--------------------------------------+------+--------+----------+--------------------------+---------+
| fcfd8081-eb72-435a-8278-b611b0a91ac4 | vm1  | ERROR  |          | cirros-0.5.2-x86_64-disk | m1.tiny |
+--------------------------------------+------+--------+----------+--------------------------+---------+
```
One can use `openstack server show -f yaml vm1`...

The error is dreaded "failed binding to port" which basically means that any kind of error occurred on Neutron
...

This is problem:
```
openstack network agent list

# empty
```


