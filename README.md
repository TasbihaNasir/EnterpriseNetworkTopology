# Enterprise-Grade BGP/OSPF Network Simulation

A hierarchical multi-AS network implementation demonstrating advanced BGP routing, OSPF integration, and inter-domain connectivity using containerized routers.

## 🎯 Project Overview

This project implements two geographically distributed campus networks (East and North) with:
- **Multi-area OSPF** for internal routing
- **iBGP** full mesh in backbone (Area 0)
- **eBGP** for inter-AS communication
- **Inter-network connectivity** between East and North campuses
- **ISP connectivity** with default route injection

### Key Features
-  Hierarchical 3-tier network design (Core, Distribution, Access)
-  Multiple Autonomous Systems (AS 65000-65104)
-  OSPF Areas 0, 1, 2 with proper ABR configuration
-  BGP route propagation and next-hop-self implementation
-  Containerized deployment using ContainerLab
-  FRRouting (FRR) for production-grade routing

---

## 🏗️ Network Architecture

### East Network Topology
```
┌────────────────────────────────────────┐
│      BACKBONE (Area 0 - AS 65000)      │
│         iBGP Full Mesh                 │
│    R1 ←→ R2 ←→ R3 ←→ R4                │
└──┬────┬─────┬─────┬────────────────────┘
   │    │     │     │
   eBGP eBGP  eBGP  eBGP
    ↓    ↓     ↓     ↓
   R5   R7   R10    R9
  (A1) (A2) (ISP) (Ext)
   AS    AS   AS     AS
  65001 65002 65004 65003
         │    │
        iBGP iBGP
         ↓    ↓
         R6   R8
         └─┘  └─┘
         PCs  PCs
```

### North Network Topology
```
┌────────────────────────────────────────┐
│      BACKBONE (Area 0 - AS 65100)      │
│         iBGP Full Mesh                 │
│    R1 ↔ R2 ↔ R3 ↔ R4                   │
└──┬────┬─────┬─────┬────────────────────┘
   │    │     │     │
 eBGP eBGP  eBGP  eBGP
   ↓    ↓     ↓     ↓
  R5   R7   R10    R9
(A1) (A2) (ISP) (Ext)
AS    AS   AS     AS
65101 65102 65104 65103
   │    │
 iBGP iBGP
   ↓    ↓
  R6   R8
  └─┘  └─┘
PC1-2 PC3-4
```

### Inter-Network Connection
```
East R1 (AS 65000) ←── eBGP ──→ North R1 (AS 65100)
  172.20.20.4                    172.20.20.19
```

---

## 🚀 Quick Start

### Prerequisites
```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y docker.io

# Install ContainerLab
bash -c "$(curl -sL https://get.containerlab.dev)"
```

### Deploy Networks

#### 1. Clone Repository
```bash
git clone https://github.com/yourusername/bgp-network-project.git
cd bgp-network-project
```

#### 2. Deploy East Network
```bash
cd FINAL_EAST
sudo containerlab deploy -t topology-east.yaml
```

#### 3. Deploy North Network
```bash
cd ../FINAL_NORTH
sudo containerlab deploy -t topology-north.yaml
```

#### 4. Wait for Convergence
```bash
# Wait 60 seconds for BGP/OSPF to converge
sleep 60
```

---

## 🔍 Verification & Testing

### Check OSPF Status
```bash
# Verify OSPF neighbors (Backbone routers)
docker exec -it clab-campus-east-ebgp-r1 vtysh -c "show ip ospf neighbor"
docker exec -it clab-campus-north-r1 vtysh -c "show ip ospf neighbor"
```

**Expected Output:**
```
Neighbor ID     State           Interface
10.0.2.2        Full/DR         eth1
10.0.3.3        Full/DR         eth2
10.0.4.4        Full/DR         eth3
```

### Check BGP Status
```bash
# iBGP + eBGP summary
docker exec -it clab-campus-east-ebgp-r1 vtysh -c "show ip bgp summary"
docker exec -it clab-campus-north-r1 vtysh -c "show ip bgp summary"
```

**Expected:** All neighbors showing "Established" state

### Verify Inter-Network Connectivity
```bash
# East can see North routes
docker exec -it clab-campus-east-ebgp-r1 vtysh -c "show ip bgp" | grep "65100"

# North can see East routes
docker exec -it clab-campus-north-r1 vtysh -c "show ip bgp" | grep "65000"

# Ping test between networks
docker exec -it clab-campus-east-ebgp-r1 ping -c 3 10.0.2.2
```

### Check Routing Table
```bash
# View learned routes
docker exec -it clab-campus-east-ebgp-r1 vtysh -c "show ip route bgp"
```

---

## 📊 Network Details

### AS Number Allocation

| Network | Area | AS Number | Role |
|---------|------|-----------|------|
| East Core | 0 | 65000 | Backbone (iBGP) |
| East Area 1 | 1 | 65001 | Distribution |
| East Area 2 | 2 | 65002 | Distribution |
| External R9 | - | 65003 | External peer |
| ISP (East) | - | 65004 | Internet gateway |
| North Core | 0 | 65100 | Backbone (iBGP) |
| North Area 1 | 1 | 65101 | Distribution |
| North Area 2 | 2 | 65102 | Distribution |
| External R9 | - | 65103 | External peer |
| ISP (North) | - | 65104 | Internet gateway |

### IP Addressing Scheme

**East Network:**
- Backbone: 10.0.X.X/30 (point-to-point)
- Loopbacks: 10.0.X.X/32
- Area 1: 10.0.56.0/30, 192.168.1.0/24, 192.168.2.0/24
- Area 2: 10.0.78.0/30
- External: 172.20.20.0/30

**North Network:**
- Backbone: 10.0.X.0/24 (broadcast)
- Loopbacks: 10.0.X.X/32
- Area 1: 10.0.56.0/24, 192.168.1.0/24, 192.168.2.0/24
- Area 2: 10.0.78.0/24
- External: 172.20.20.0/30

---

## 🛠️ Configuration Highlights

### iBGP Configuration (Backbone)
```bash
router bgp 65000
 bgp router-id 10.0.1.1
 no bgp ebgp-requires-policy
 
 # iBGP neighbors using loopbacks
 neighbor 10.0.2.2 remote-as 65000
 neighbor 10.0.2.2 update-source lo
 neighbor 10.0.3.3 remote-as 65000
 neighbor 10.0.3.3 update-source lo
 
 address-family ipv4 unicast
  redistribute ospf
  neighbor 10.0.2.2 next-hop-self
  neighbor 10.0.3.3 next-hop-self
 exit-address-family
```

### eBGP Configuration (AS Boundary)
```bash
router bgp 65000
 # eBGP to external AS
 neighbor 10.0.15.2 remote-as 65001
 
 address-family ipv4 unicast
  neighbor 10.0.15.2 activate
 exit-address-family
```

### OSPF Configuration
```bash
router ospf
 ospf router-id 10.0.1.1
 network 10.0.1.1/32 area 0
 network 10.0.12.0/30 area 0
 network 10.0.13.0/30 area 0
```

---

## 🧪 Advanced Testing

### Traceroute Between Networks
```bash
# Trace path from East to North
docker exec -it clab-campus-east-ebgp-r1 traceroute 10.0.6.6
```

### BGP Path Analysis
```bash
# View BGP path attributes
docker exec -it clab-campus-east-ebgp-r1 vtysh -c "show ip bgp 192.168.1.0/24"
```

### Route Propagation Test
```bash
# Check how routes propagate through ASes
docker exec -it clab-campus-east-ebgp-r1 vtysh -c "show ip bgp regexp 65100"
```

---

## 🐛 Troubleshooting

### BGP Neighbors Not Establishing
```bash
# Check interface status
docker exec -it clab-campus-east-ebgp-r1 ip addr show

# Verify BGP configuration
docker exec -it clab-campus-east-ebgp-r1 vtysh -c "show run | section router bgp"

# Check BGP logs
docker exec -it clab-campus-east-ebgp-r1 vtysh -c "show log" | grep BGP
```

### OSPF Not Forming Adjacencies
```bash
# Check OSPF interface configuration
docker exec -it clab-campus-east-ebgp-r1 vtysh -c "show ip ospf interface"

# Verify OSPF database
docker exec -it clab-campus-east-ebgp-r1 vtysh -c "show ip ospf database"
```

### Clear BGP Sessions
```bash
# Soft reset BGP
docker exec -it clab-campus-east-ebgp-r1 vtysh -c "clear ip bgp * soft"

# Hard reset BGP
docker exec -it clab-campus-east-ebgp-r1 vtysh -c "clear ip bgp *"
```

---

## 🧹 Cleanup

### Destroy Networks
```bash
# Destroy East network
cd FINAL_EAST
sudo containerlab destroy -t topology-east.yaml

# Destroy North network
cd ../FINAL_NORTH
sudo containerlab destroy -t topology-north.yaml
```

### Remove All Containers
```bash
docker ps -a | grep clab | awk '{print $1}' | xargs docker rm -f
```

---

## 📁 Project Structure

```
bgp-network-project/
├── FINAL_EAST/
│   ├── topology-east.yaml          # East network topology
│   ├── r1/
│   │   ├── frr.conf               # FRR configuration
│   │   ├── daemons                # Enabled daemons
│   │   └── vtysh.conf             # vtysh settings
│   ├── r2/ ... r10/               # Similar structure
│   └── README.md
├── FINAL_NORTH/
│   ├── topology-north.yaml        # North network topology
│   ├── r1/ ... r10/               # Router configs
│   └── README.md
├── docs/
│   ├── network-design.md          # Design documentation
│   ├── verification.md            # Testing procedures
│   └── troubleshooting.md         # Common issues
└── README.md                      # This file
```

---

## 🎓 Learning Outcomes

This project demonstrates proficiency in:

- **Network Design**: Hierarchical topology with proper segmentation
- **Routing Protocols**: OSPF (link-state) and BGP (path-vector)
- **Protocol Integration**: IGP/EGP interaction and redistribution
- **Containerization**: Docker and ContainerLab for network simulation
- **Linux Networking**: Interface configuration, routing tables
- **DevOps Practices**: Infrastructure as Code, version control
- **Troubleshooting**: Systematic debugging of routing issues

---

## 🚀 Future Enhancements

### Short-term Improvements
-  Add MPLS/VPN functionality
-  Implement BGP route filtering with route-maps
-  Add BFD for faster convergence
-  Implement IPv6 dual-stack
-  Add monitoring with Prometheus/Grafana

### Medium-term Goals
-  **Cloud Deployment**: Migrate to AWS/GCP/Azure
  - Deploy routers as EC2/Compute Engine instances
  - Use VPC peering for inter-network connectivity
  - Implement BGP over VPN tunnels
-  Add load balancing with ECMP
-  Implement BGP communities for traffic engineering
-  Add firewall policies between ASes
---


## 📚 References & Resources

### Documentation
- [FRRouting Documentation](https://docs.frrouting.org/)
- [ContainerLab Guide](https://containerlab.dev/)
- [RFC 4271 - BGP-4](https://datatracker.ietf.org/doc/html/rfc4271)
- [RFC 2328 - OSPF v2](https://datatracker.ietf.org/doc/html/rfc2328)

### Learning Resources
- [BGP Best Practices](https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html)
- [OSPF Design Guide](https://www.cisco.com/c/en/us/support/docs/ip/open-shortest-path-first-ospf/7039-1.html)

---

