# MacVTap OpenStack install


Simples possible DevStack configuration where VM is directly attached to main bridge.


> WARNING!
>
> MacVTap is terribly broken in DevStack because there no file `macvtap_agent` under
> `lib/neutron_plugins` of DevStack source - so it can't work!
>
> I plan to use `linuxbridge` instead

Requirements
* Ubuntu 22.04 LTS
* main network card configure as Bridge with static IP address. See `etc` in this folder for example.

# Setup

You need Ubuntu 22.04 LTS with properly configured static IP address via Bridge (so OpenStack VMs
can attach to it to get network access). Also both `hostname` and `hostname -f` must properly
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
This is your host IP address: 192.168.0.7
This is your host IPv6 address: 2a02:d8a0:a:1c29:e3:a0ff:fe4e:339c
Horizon is now available at http://192.168.0.7/dashboard
Keystone is serving at http://192.168.0.7/identity/
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
Under openstack shell query environment:
```
# We need at least one image

(openstack) image list
+--------------------------------------+--------------------------+--------+
| ID                                   | Name                     | Status |
+--------------------------------------+--------------------------+--------+
| d5db14f9-7521-47a9-83b9-304ae652a5dc | cirros-0.5.2-x86_64-disk | active |
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

openstack) hypervisor list
+--------------------------------------+---------------------+-----------------+-------------+-------+
| ID                                   | Hypervisor Hostname | Hypervisor Type | Host IP     | State |
+--------------------------------------+---------------------+-----------------+-------------+-------+
| cc6867f4-471e-4a98-abf3-3a258dd27e07 | devstack2           | QEMU            | 192.168.0.7 | up    |
+--------------------------------------+---------------------+-----------------+-------------+-------+


# we need at least one network

(openstack) network list
+--------------------------------------+---------+--------------------------------------+
| ID                                   | Name    | Subnets                              |
+--------------------------------------+---------+--------------------------------------+
| 52d880d6-d558-45db-a7a8-0aa910252576 | public  | 3208b2ea-8d11-442f-af5b-5d6b6a4779a7 |
| aaacf2d0-af92-4970-bd69-22304389f69f | default | 5c22031e-f5f5-4c64-8b5f-8543bf2631e3 |
+--------------------------------------+---------+--------------------------------------+

# Our network is "default"
# SHow subnets:

(openstack) subnet list
--------------------------------------+---------------+--------------------------------------+----------------+
| ID                                   | Name          | Network                              | Subnet         |
+--------------------------------------+---------------+--------------------------------------+----------------+
| 3208b2ea-8d11-442f-af5b-5d6b6a4779a7 | public-subnet | 52d880d6-d558-45db-a7a8-0aa910252576 | 172.24.4.0/24  |
| 5c22031e-f5f5-4c64-8b5f-8543bf2631e3 | provider_net  | aaacf2d0-af92-4970-bd69-22304389f69f | 192.168.0.0/24 |
+--------------------------------------+---------------+--------------------------------------+----------------+


# Double check our subnet "provider_net"

(openstack) subnet show provider_net
+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| allocation_pools     | 192.168.0.2-192.168.0.254            |
| cidr                 | 192.168.0.0/24                       |
| created_at           | 2023-11-25T14:09:40Z                 |
| description          |                                      |
| dns_nameservers      |                                      |
| dns_publish_fixed_ip | None                                 |
| enable_dhcp          | True                                 |
| gateway_ip           | 192.168.0.1                          |
| host_routes          |                                      |
| id                   | 5c22031e-f5f5-4c64-8b5f-8543bf2631e3 |
| ip_version           | 4                                    |
| ipv6_address_mode    | None                                 |
| ipv6_ra_mode         | None                                 |
| name                 | provider_net                         |
| network_id           | aaacf2d0-af92-4970-bd69-22304389f69f |
| project_id           | f90724b164ad4e4fb925708cabd1580b     |
| revision_number      | 0                                    |
| segment_id           | None                                 |
| service_types        |                                      |
| subnetpool_id        | None                                 |
| tags                 |                                      |
| updated_at           | 2023-11-25T14:09:40Z                 |
+----------------------+--------------------------------------+

# We need to corrections
# 1. Disable DHCP (otherwise there will be collision with my Network's DHCP0;
subnet set --no-dhcp provider_net

# 2. Restrict allocated addresses to avoid collisions with other devices on main network
#    in my case:
(openstack) subnet set --no-allocation-pool provider_net
(openstack) subnet set --allocation-pool start=192.168.0.60,end=192.168.0.99 provider_net

# now check new settings
(openstack) subnet show provider_net
+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| allocation_pools     | 192.168.0.60-192.168.0.99            |
| cidr                 | 192.168.0.0/24                       |
| created_at           | 2023-11-25T14:09:40Z                 |
| description          |                                      |
| dns_nameservers      |                                      |
| dns_publish_fixed_ip | None                                 |
| enable_dhcp          | False                                |
| gateway_ip           | 192.168.0.1                          |
| host_routes          |                                      |
| id                   | 5c22031e-f5f5-4c64-8b5f-8543bf2631e3 |
| ip_version           | 4                                    |
| ipv6_address_mode    | None                                 |
| ipv6_ra_mode         | None                                 |
| name                 | provider_net                         |
| network_id           | aaacf2d0-af92-4970-bd69-22304389f69f |
| project_id           | f90724b164ad4e4fb925708cabd1580b     |
| revision_number      | 3                                    |
| segment_id           | None                                 |
| service_types        |                                      |
| subnetpool_id        | None                                 |
| tags                 |                                      |
| updated_at           | 2023-11-25T15:01:51Z                 |
+----------------------+--------------------------------------+
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
openstack server create --network default --flavor m1.tiny \
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
openstack network agent listo

# empty
```

Serious issue:
- `~/projects/devstack/lib/neutron_plugins/` does not contain file `macvtap` which is necessary to 
   configure properly....

