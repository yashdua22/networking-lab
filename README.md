# networking-lab
# 🌐 Two-Network Lab with FRR Routing

> A hands-on networking lab: two **isolated networks** connected by a real **FRRouting (FRR)** router, built as code with [Containerlab](https://containerlab.dev/) and Docker. Watch a single packet cross the boundary — and prove *why* it needs a router to do it.



## 📖 What this project is

Two separate networks can't talk to each other on their own — they're like two neighbourhoods with different street-naming systems. The only thing that can ferry messages between them is a **router** that has one foot in each network.

This lab builds exactly that, from scratch:

- 🏠 **Host 1** — lives in Network 1 (`10.0.1.0/24`), acts as the client
- 🏠 **Host 2** — lives in Network 2 (`10.0.2.0/24`), runs Nginx as a web server
- 🔀 **Router** — an FRR router sitting between them, one interface in each network

Then it proves routing works with `ping` and `curl`, breaks it on purpose by disabling IP forwarding, and analyzes the live traffic in **Wireshark** down to the TTL field.

---

## 🗺️ Topology

```
   Network 1 (10.0.1.0/24)              Network 2 (10.0.2.0/24)
 ┌───────────────────────┐          ┌───────────────────────┐
 │                       │          │                       │
 │   host1               │          │              host2    │
 │   10.0.1.10           │          │          10.0.2.10    │
 │        │              │          │              │        │
 └────────┼──────────────┘          └──────────────┼────────┘
          │ eth1                              eth2  │
          │                                         │
          │        ┌───────────────────┐           │
          └────────┤  router (FRR)      ├───────────┘
            eth1    │  eth1: 10.0.1.1   │   eth2
                    │  eth2: 10.0.2.1   │
                    └───────────────────┘
```

The router has a live address in **both** networks (`10.0.1.1` and `10.0.2.1`), which is exactly why it can pass traffic between them.

---

## 🧰 Tech stack

| Tool | Role |
|------|------|
| 🐳 **Docker Engine** | Runs every node as a container |
| 📦 **Containerlab** | Defines & spins up the whole topology from one YAML file |
| 🔀 **FRRouting (FRR)** | The actual router — same open-source stack used on real network gear |
| 🛠️ **network-multitool** | Lightweight hosts preloaded with ping, curl, tcpdump, Nginx |
| 🦈 **Wireshark** | Packet-level analysis of the captured traffic |

---

## 📂 Project structure

```
two-nets-lab/
├── two-nets.clab.yml      # the topology — what connects to what
└── router/
    ├── daemons            # which FRR routing daemons to enable
    └── frr.conf           # the router's interface addresses & config
```

---

## 📄 The files

<details>
<summary><strong>two-nets.clab.yml</strong> — the topology</summary>

```yaml
name: two-nets

topology:
  nodes:
    router:
      kind: linux
      image: frrouting/frr:latest
      sysctls:
        net.ipv4.ip_forward: 1
      binds:
        - router/daemons:/etc/frr/daemons
        - router/frr.conf:/etc/frr/frr.conf

    host1:
      kind: linux
      image: wbitt/network-multitool:latest
      exec:
        - ip addr add 10.0.1.10/24 dev eth1
        - ip route replace default via 10.0.1.1

    host2:
      kind: linux
      image: wbitt/network-multitool:latest
      exec:
        - ip addr add 10.0.2.10/24 dev eth1
        - ip route replace default via 10.0.2.1

  links:
    - endpoints: ["host1:eth1", "router:eth1"]
    - endpoints: ["host2:eth1", "router:eth2"]
```
</details>

<details>
<summary><strong>router/frr.conf</strong> — the router's brain</summary>

```
frr version 8.4
frr defaults traditional
hostname router
no ipv6 forwarding
!
interface eth1
 ip address 10.0.1.1/24
 description link-to-network-1
!
interface eth2
 ip address 10.0.2.1/24
 description link-to-network-2
!
line vty
!
```
</details>

<details>
<summary><strong>router/daemons</strong> — which daemons to start</summary>

```
zebra=yes
mgmtd=yes
bgpd=no
ospfd=no
ripd=no
ospf6d=no
ripngd=no
isisd=no
pimd=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
bfdd=no
fabricd=no
vrrpd=no
pathd=no
```
</details>

---

## 🚀 Setup & run

> Built and tested on **WSL2 (Ubuntu)** with native Docker Engine.
> ⚠️ Use native Docker Engine inside Ubuntu, **not** Docker Desktop's WSL backend — Containerlab needs direct access to the Linux network stack to build the virtual links.

### 1️⃣ Start Docker

```bash
sudo service docker start
```

### 2️⃣ Deploy the lab

```bash
sudo containerlab deploy -t two-nets.clab.yml
```

In ~30 seconds you'll see a table with `clab-two-nets-router`, `clab-two-nets-host1`, and `clab-two-nets-host2`. 🎉

---

## ✅ Testing it works

### 🔗 Cross the boundary

```bash
docker exec -it clab-two-nets-host1 bash
ping -c 3 10.0.2.10          # ping host2 across the router
curl http://10.0.2.10        # fetch host2's web page through the router
```

The ping returns with **`ttl=63`** — proof it passed through exactly one router (TTL starts at 64, each router subtracts 1). 🔑

### 🔍 Inspect the router like an engineer

```bash
docker exec -it clab-two-nets-router vtysh
```
```
show ip route          # both networks show as directly connected (C>*)
show interface brief   # eth1 & eth2 up, with their addresses
```

---

## 💥 Break it on purpose (the real lesson)

A router only forwards traffic because IP forwarding is switched on. Flip it off and the two networks go deaf to each other:

```bash
# turn forwarding OFF — ping from host1 now FAILS
docker exec clab-two-nets-router sysctl -w net.ipv4.ip_forward=0

# turn it back ON — ping works again
docker exec clab-two-nets-router sysctl -w net.ipv4.ip_forward=1
```

That one toggle *is* what a router does, felt in your hands. 🧠

---

## 🦈 Packet analysis with Wireshark

### Capture traffic

```bash
# terminal 1 — sniff on host2 and save to a file
docker exec clab-two-nets-host2 tcpdump -i eth1 -w /tmp/capture.pcap

# terminal 2 — generate traffic
docker exec -it clab-two-nets-host1 ping -c 5 10.0.2.10

# stop terminal 1 with Ctrl+C, then copy the file out
docker cp clab-two-nets-host2:/tmp/capture.pcap ~/capture.pcap
```

### Open in Wireshark

Copy the file to Windows and open it, then filter with:

```
icmp        # see the ping request/reply pairs
http        # see the curl GET and 200 OK response
```

**What to look for:** each `request` leaves `host1` with `ttl=63` (already decremented by the router), while each `reply` from `host2` is caught fresh at `ttl=64`. That contrast is routing, made visible. 👀

---

## 🧹 Clean up

```bash
sudo containerlab destroy -t two-nets.clab.yml --cleanup
```

---

## 📚 What I learned

- 🔀 How a router physically moves traffic between isolated networks
- 🧩 Configuring a real FRR router (interfaces, addresses, daemons)
- 🔑 Why TTL decrements — and how to use it to count hops
- ⚙️ The role of `net.ipv4.ip_forward` in making a box a router
- 🦈 Reading packets layer by layer (Ethernet → IP → ICMP) in Wireshark
- 🐞 Debugging a real Docker/WSL networking issue along the way

---

## 🔮 Next steps

- ➕ Add a second FRR router and run **OSPF** so routers *learn* routes dynamically instead of relying on directly-connected networks
- 🔒 Add firewall rules between the networks
- 📊 Add a third network and observe multi-hop TTL decrements

---

### 🏷️ Tags
`#Networking` `#DevOps` `#Containerlab` `#FRRouting` `#Docker` `#Wireshark` `#Linux`
