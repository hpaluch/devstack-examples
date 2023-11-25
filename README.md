# DevStack setup examples

My personal setup examples for devstack
- https://docs.openstack.org/devstack/latest/index.html

WORK IN PROGRESS!

See also my wiki page for details:
- https://github.com/hpaluch/hpaluch.github.io/wiki/DevStack-Quick-Start

All examples are using Ubuntu 22.04 LTS (because it is used by OpenStack developers and is thus
most likely to work).

## Conservative

Conservative setup, that just adds support for Older CPUs
is under [conservative/](conservative/) folder.


## MacVTap

> MacVTap is UNUSABLE on DevStack because of missing Agent support under `lib/neutron_plugins`!!!
> This text is here kept for reference only...

MacVTap is most simple interface that connects VMs directly to Bridge interfaces. It is similar
to way how other solutions (Proxmox VE) works.

Project files are under [macvtap/](macvtap/).

See https://docs.openstack.org/devstack/latest/guides/neutron.html#single-node-with-provider-networks-using-config-drive-and-external-l3-dhcp
for *outdated* configuration.


