# linux-ns-networking-3

## Connecting a container network namespace to root network namespace

Let's create a custom network namespace `ns0` and a bridge `br0`.
In Linux networking, a bridge is a virtual network device that connects multiple network interfaces, allowing them to function as a single logical network.

```bash
sudo ip netns add ns0
sudo ip link add br0 type bridge
```

![ns0-br0](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/3ef5b56c-5e5f-4a06-83dd-94d0d1ad6b4b.png)

## Configure a bridge interface

A new device, the `br0` bridge interface, has been created, but it's now in a `DOWN` state. Let's assign ip address and turn it into `UP` state.

```bash
sudo ip link set br0 up
sudo ip addr add 192.168.0.1/16 dev br0
```

![br0 setup](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/5c391867-6f00-4845-82fd-900ab0d18dea.png)

Now let's verify whether br0 is able to receive the packet or not.

![ping to br0](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/9c7d50b8-e48e-4dd5-9a9f-1bef9c53d289.png)

## Configure virtual ethernet cable

It's time to set up a virtual Ethernet cable. One cable hand will be configured as a nic card in the ns0 namespace, while the other hand will be configured in the br0 interface.

```bash
sudo ip link add veth0 type veth peer name ceth0
sudo ip link set ceth0 netns ns0
sudo ip link set veth0 master br0
```

Both end of this cable is now in `DOWN` state. Let's turn into `UP` state

```bash
sudo ip netns exec ns0 ip link set ceth0 up
sudo ip link set veth br0
```

## Configure ns0 namespace

We need to assign an ip address to ceth0 and turn loopback interface into `UP` state.

```bash
sudo ip link set lo up
sudo ip addr add 192.168.0.2/16 dev ceth0
```
## Namespace ns0 to root ns Communication

Let's check the Ip address assigned to primary ethernet interface of host machine.

```bash
ip addr show 
```

![ens3](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/d4a85484-1c75-4103-8d81-152a23884568.png)

Now, ping to this ip address

```bash
sudo ip netns exec ns0 bash
ping 10.0.0.25
```

![output](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/88232a86-b214-4028-a151-468858ae09db.png)

It says network in unreachable. So, something is not okay. Let's check the route table.
```bash
route
```

The output may look like.

```bash
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.0.0     0.0.0.0         255.255.0.0     U     0      0        0 ceth0
```

This routing table entry indicates that any destination IP address within the 192.168.0.0/16 network should be reached directly through the ceth0 interface, without the need for a specific gateway.

So we need to add a Default Gateway in the route table.

```bash
ip netns exec ns0 bash
ip route add default via 192.168.0.1
```

![route added](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/cf1b99fe-a9eb-40b5-a2ff-16b62920c822.png)

Now we are good to go! Let's ping again.

```bash
sudo ip netns exec ns0 bash
ping 10.0.0.25 -c 5
```

![ping-pong](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/00a7eba6-650d-47d2-9bc8-2adc2dd25a09.png)