# Running CoreOS on HP Public Cloud (hpcloud.com) 

This page was derived from, and is meant to be used in conjunction with, the CoreOS OpenStack [guide][coreos-guide].

## CoreOS Image

The stable CoreOS image is already registered as a partner image in glance with the name ```CoreOS```.

To use an image from a different channel (beta or alpha), follow the directions in the CoreOS [guide][coreos-guide].

[coreos-guide]: https://coreos.com/docs/running-coreos/platforms/openstack/

## CoreOS Security Group

The CoreOS etcd component is responsible for cluster discovery and provides a distributed lock manager (like Google Chubby).  `etcd` uses port `7001` for peer coordination and port `4001` for as a user REST API.  Both ports need to be enabled.  You can do this by either adding rules for these ports to the `default` security group, or create a new security group called `coreos`:

```sh
nova secgroup-create coreos "coreos security group with rules for etcd"
nova secgroup-add-rule coreos icmp -1 -1 0.0.0.0/0
nova secgroup-add-rule coreos tcp 22 22 0.0.0.0/0
nova secgroup-add-rule coreos tcp 4001 4001 0.0.0.0/0
nova secgroup-add-rule coreos tcp 7001 7001 0.0.0.0/0
```

## Cloud-Config

Create a cloud-config file named cloud-config.yaml.  The most common cloud-config for OpenStack looks like:

```yaml
#cloud-config

coreos:
  etcd:
    # generate a new token for each unique cluster from https://discovery.etcd.io/new?size=3
    # specify the initial size of your cluster with ?size=X
    discovery: https://discovery.etcd.io/<token>
    # multi-region and multi-cloud deployments need to use $public_ipv4
    addr: $public_ipv4:4001
    peer-addr: $public_ipv4:7001
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
ssh_authorized_keys:
  # include one or more SSH public keys
  - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC0g+ZTxC7weoIJLUafOgrm+h...
```

The `$private_ipv4` and `$public_ipv4` substitution variables are fully supported in cloud-config. For HP Public Cloud, make sure to use the $public_ipv4 value.

NOTE in the above: You must replace two fields: the `https://discovery.etcd.io/<token>` URL with a real discovery token obtained as listed above, and the SSH public key with your own.

Troubleshooting tip: EVERY TIME you create a new cluster, also create a new cluster token, especially if you're reusing the same floating IP's.

## Get Network ID

Find the ID for your configured network as follows:

```
nova network-list
+--------------------------------------+---------+------+
| ID                                   | Label   | Cidr |
+--------------------------------------+---------+------+
| f54b48c7-34fc-4828-8ee9-21b623c7b8f9 | public  | -    |
| 5b9c5ef6-28b9-4781-ac18-d7d86765fd38 | private | -    |
+--------------------------------------+---------+------+
```

You can use the private Network ID.  We will assign floating IP's later.

## Launch Cluster

Boot the machines with the `nova` CLI, referencing the image ID from the import step above and your `cloud-config.yaml`:

```sh
nova boot \
--user-data ./cloud-config.yaml \
--image CoreOS \
--key-name coreos \
--flavor m1.medium \
--num-instances 3 \
--security-groups coreos \
--nic net-id=5b9c5ef6-28b9-4781-ac18-d7d86765fd38 \
coreos
```

NOTE: Use the same SSH key in the --key-name option as you used in the cloud-config file.

NOTE: substitute the Network ID that you obtained in the previous step.

## Floating IP's

Now you'll need to assign floating IP's to each instance.  You can create new floating IP's with the following:

```nova floating-ip-create```

To associate floating IP's with each instance, do the following:

```sh
nova floating-ip-associate coreos-1 15.126.y.a
nova floating-ip-associate coreos-2 15.126.y.b
nova floating-ip-associate coreos-3 15.126.y.c
```

Your first CoreOS cluster should now be running. The only thing left to do is
find an IP and SSH in.

```sh
$ nova list
+--------------------------------------+-----------------+--------+------------+-------------+-------------------------------+
| ID                                   | Name            | Status | Task State | Power State | Networks                      |
+--------------------------------------+-----------------+--------+------------+-------------+-------------------------------+
| a1df1d98-622f-4f3b-adef-cb32f3e2a94d | coreos-1        | ACTIVE | None       | Running     | private=10.0.0.3, 15.126.y.a  |
| db13c6a7-a474-40ff-906e-2447cbf89440 | coreos-2        | ACTIVE | None       | Running     | private=10.0.0.4, 15.126.y.b  |
| f70b739d-9ad8-4b0b-bb74-4d715205ff0b | coreos-3        | ACTIVE | None       | Running     | private=10.0.0.5, 15.126.y.c  |
+--------------------------------------+-----------------+--------+------------+-------------+-------------------------------+
```

Finally SSH into an instance, note that the user is `core`:

```sh
$ chmod 400 core.pem
$ ssh -i core.pem core@15.126.y.a
   ______                ____  _____
  / ____/___  ________  / __ \/ ___/
 / /   / __ \/ ___/ _ \/ / / /\__ \
/ /___/ /_/ / /  /  __/ /_/ /___/ /
\____/\____/_/   \___/\____//____/

core@15.126.y.a ~ $
```

## Troubleshooting

Find out if etcd is running by issuing a ```ps -ef |grep etcd```

If not, the following command may give some context: ```journalctl -u etcd```

## Adding More Machines

Adding new instances to the cluster is as easy as launching more with the same 
cloud-config. New instances will join the cluster assuming they can communicate 
with the others.

Example:

```sh
nova boot \
--user-data ./cloud-config.yaml \
--image CoreOS \
--key-name coreos \
--flavor m1.medium \
--security-groups coreos \
coreos
```

## Multiple Clusters

If you would like to create multiple clusters you'll need to generate and use a
new discovery token. Change the token value on the etcd discovery parameter in the cloud-config, and boot new instances.

## Using CoreOS

Now that you have instances booted it is time to play around.
Check out the [CoreOS Quickstart][coreos-quickstart].

[coreos-quickstart]: https://coreos.com/docs/quickstart/
