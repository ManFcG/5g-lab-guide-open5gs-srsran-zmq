# 5G Network Simulation Guide: Open5GS 5GC & srsRAN with ZeroMQ

This guide details the deployment of a simulated 5G network using **Open5GS** (Core) and **srsRAN** (RAN/UE) communicating via ZeroMQ. This setup allows for end-to-end packet transmission testing in a virtualized environment.

## 1. System Overview

The architecture consists of four Virtual Machines simulating the Core Network, Radio Access Network, and User Equipment.

### 1.1 Virtual Machine Specifications

| Role | VM ID | IP Address | OS | Resources (CPU/RAM/HDD) |
|:---|:---|:---|:---|:---|
| **5GC C-Plane** | VM1 | 172.16.0.50/24 | Ubuntu 24.04 | 1 vCPU / 2 GB / 20 GB |
| **5GC U-Plane** | VM2 | 172.16.0.51/24 | Ubuntu 24.04 | 1 vCPU / 1 GB / 20 GB |
| **gNodeB (RAN)**| VM3 | 172.16.0.60/24 | Ubuntu 24.04 | 4 vCPUs / 4 GB / 10 GB |
| **NR-UE** | VM4 | 172.16.0.61/24 | Ubuntu 22.04 | 1 vCPU / 2 GB / 10 GB |

### 1.2 Network Parameters

**Core Network Settings:**
- **AMF IP:** `172.16.0.50` (SBI: `127.0.0.5`)
- **SMF IP:** `172.16.0.50` (SBI: `127.0.0.4`)
- **S-NSSAI:** SST: 1, SD: 0x000001

**Subscriber Data:**
- **IMSI:** `001010000000000`
- **Key (K):** `465B5CE8B199B49FAA5F0A2EE238A6BC`
- **OPc:** `E8ED289DEBA952E4283B54E88E6183CA`
- **DNN:** `internet`

**Data Network (DN):**
- **Subnet:** `10.45.0.0/16`
- **S-NSSAI:** SST: 1, SD: 0x000001
- **Tunnel Interface (DN):** `ogstun`
- **Tunnel Interface (UE):** `tun_srsue`

**Radio Configuration:**
- **PLMN:** 00101
- **TAC:** 1
- **gNB ID:** 0x19B
- **MCC:** 001
- **MNC:** 01

> **Note:** I registered the subscriber information via the Open5GS WebUI.

## 2. Installation Guide

Follow these steps to build the necessary components on their respective VMs.

The following guides and documentation were used as the basis for this deployment:
- Open5GS v2.7.6 (30.11.2025) - [Documentation](https://open5gs.org/open5gs/docs/guide/02-building-open5gs-from-sources/), [Open5GS v2.7.6](https://github.com/open5gs/open5gs/releases/tag/v2.7.6)
- Open5GS WebUI - [Documentation](https://open5gs.org/open5gs/docs/guide/01-quickstart/), [Guide](https://github.com/s5uishida/open5gs_install_mongodb_webui)
- srsRAN Project (RAN) v25.04 (07.07.2025) - [Guide](https://github.com/s5uishida/build_srsran_5g_zmq), [srsRAN Project 25.04](https://github.com/srsran/srsRAN_Project/releases/tag/release_25_04)
- srsRAN 4G (UE) (22.10.2025) - [Guide](https://github.com/s5uishida/build_srsran_4g_zmq_disable_rf_plugins), [srsRAN 4G](https://github.com/srsran/srsRAN_4G/)

### 2.1 Open5GS (C-Plane & U-Plane)

**Prerequisites (MongoDB for C-Plane):**
```bash
sudo apt update
sudo apt install gnupg
curl -fsSL https://pgp.mongodb.com/server-8.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/8.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

sudo apt update
sudo apt install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod
```

**Build Dependencies:**
```bash
sudo apt install python3-pip python3-setuptools python3-wheel ninja-build build-essential flex bison git cmake libsctp-dev libgnutls28-dev libgcrypt-dev libssl-dev libmongoc-dev libbson-dev libyaml-dev libnghttp2-dev libmicrohttpd-dev libcurl4-gnutls-dev libnghttp2-dev libtins-dev libtalloc-dev meson
```
```bash
if apt-cache show libidn-dev > /dev/null 2>&1; then
    sudo apt-get install -y --no-install-recommends libidn-dev
else
    sudo apt-get install -y --no-install-recommends libidn11-dev
fi
```

**Compile Open5GS:**
```bash
git clone https://github.com/open5gs/open5gs
cd open5gs
meson build --prefix=`pwd`/install
ninja -C build
ninja -C build install
```

**Install Open5GS WebUI:**

1. Install Node.js:
```bash
sudo apt install -y ca-certificates curl gnupg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg

# Create deb repository
NODE_MAJOR=20
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list

# Run Update and Install
sudo apt update
sudo apt install nodejs -y
```

2. Install WebUI Package:
```bash
curl -fsSL https://open5gs.org/open5gs/assets/webui/install | sudo -E bash -
```

3. Configure Binding Address (Optional):
If you need to bind the WebUI to a specific IP (e.g., `172.16.0.50`), edit `/lib/systemd/system/open5gs-webui.service`:
```diff
 WorkingDirectory=/usr/lib/node_modules/open5gs
 Environment=NODE_ENV=production
+Environment=HOSTNAME=172.16.0.50
+Environment=PORT=3000
 ExecStart=/usr/bin/node server/index.js
 Restart=always
 RestartSec=2
```

4. Restart Service:
```bash
sudo systemctl daemon-reload
sudo systemctl restart open5gs-webui
```

5. Access WebUI:
- **URL:** `http://172.16.0.50:3000/`
- **Username:** `admin`
- **Password:** `1423`


### 2.2 srsRAN Project (gNodeB)

**Dependencies:**
```bash
apt install cmake make gcc g++ pkg-config libfftw3-dev libmbedtls-dev libsctp-dev libyaml-cpp-dev libgtest-dev libzmq3-dev
```

**Build Instructions:**
```bash
sudo apt-get install unzip
wget https://github.com/srsran/srsRAN_Project/archive/refs/tags/release_25_04.zip
unzip release_25_04.zip
cd srsRAN_Project-release_25_04
mkdir build
cd build
cmake ../ -DENABLE_EXPORT=ON -DENABLE_ZEROMQ=ON
make -j`nproc`
cp $(pwd)/apps/gnb /usr/bin/gnb
cd ~
```

### 2.3 srsRAN 4G (UE)

**Dependencies:**
```bash
apt install build-essential cmake libfftw3-dev libmbedtls-dev libboost-program-options-dev libconfig++-dev libsctp-dev libzmq3-dev
```

**Build Instructions:**
```bash
git clone https://github.com/srsran/srsRAN_4G.git
cd srsRAN_4G/
mkdir build
cd build/
cmake ../ -DENABLE_EXPORT=ON -DENABLE_ZEROMQ=ON
make -j`nproc`
cp $(pwd)/srsue/src/srsue /usr/bin/srsue
cd ~
```

## 3. Configuration
Apply the following changes to the configuration files.

### 3.1 Network Setup (U-Plane VM)

Enable IP forwarding and configure NAT:
```bash
net.ipv4.ip_forward=1
```
Apply with:
```bash
sysctl -p
```
Setup TUN interface:
```bash
ip tuntap add name ogstun mode tun
ip addr add 10.45.0.1/16 dev ogstun
ip link set ogstun up

iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
```

### 3.2 Open5GS Configuration

#### C-Plane Configs

**`open5gs/install/etc/open5gs/amf.yaml`**
```diff
         - uri: http://127.0.0.200:7777
   ngap:
     server:
-      - address: 127.0.0.5
+      - address: 172.16.0.50
   metrics:
     server:
       - address: 127.0.0.5
         port: 9090
   guami:
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
       amf_id:
         region: 2
         set: 1
   tai:
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
       tac: 1
   plmn_support:
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
       s_nssai:
         - sst: 1
+          sd: 000001
   security:
     integrity_order : [ NIA2, NIA1, NIA0 ]
     ciphering_order : [ NEA0, NEA1, NEA2 ]
```
**`open5gs/install/etc/open5gs/nrf.yaml`**
```diff
 nrf:
   serving:  # 5G roaming requires PLMN in NRF
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
   sbi:
     server:
       - address: 127.0.0.10
```
**`open5gs/install/etc/open5gs/smf.yaml`**
```diff
         - uri: http://127.0.0.200:7777
   pfcp:
     server:
-      - address: 127.0.0.4
+      - address: 172.16.0.50
     client:
       upf:
-        - address: 127.0.0.7
+        - address: 172.16.0.51
+          dnn: internet
   gtpc:
     server:
       - address: 127.0.0.4
   gtpu:
     server:
-      - address: 127.0.0.4
+      - address: 172.16.0.50
   metrics:
     server:
       - address: 127.0.0.4
@@ -37,20 +38,17 @@
   session:
     - subnet: 10.45.0.0/16
       gateway: 10.45.0.1
-    - subnet: 2001:db8:cafe::/48
-      gateway: 2001:db8:cafe::1
+      dnn: internet
   dns:
     - 8.8.8.8
     - 8.8.4.4
-    - 2001:4860:4860::8888
-    - 2001:4860:4860::8844
   mtu: 1400
 #  p-cscf:
 #    - 127.0.0.1
 #    - ::1
 #  ctf:
 #    enabled: auto   # auto(default)|yes|no
-  freeDiameter: /root/open5gs/install/etc/freeDiameter/smf.conf
+#  freeDiameter: /root/open5gs/install/etc/freeDiameter/smf.conf
```
**`open5gs/install/etc/open5gs/nssf.yaml`**
```diff
         - uri: http://127.0.0.10:7777
           s_nssai:
             sst: 1
+            sd: 000001
```
#### U-Plane Configs
**`open5gs/install/etc/open5gs/upf.yaml`**
```diff
 upf:
   pfcp:
     server:
-      - address: 127.0.0.7
+      - address: 172.16.0.51
     client:
 #      smf:     #  UPF PFCP Client try to associate SMF PFCP Server
 #        - address: 127.0.0.4
   gtpu:
     server:
-      - address: 127.0.0.7
+      - address: 172.16.0.51
   session:
     - subnet: 10.45.0.0/16
       gateway: 10.45.0.1
-    - subnet: 2001:db8:cafe::/48
-      gateway: 2001:db8:cafe::1
+      dnn: internet
+      dev: ogstun
   metrics:
     server:
       - address: 127.0.0.7
```
### 3.3 RAN & UE Configuration

#### gNodeB (RAN)
[Original file](https://docs.srsran.com/projects/project/en/latest/tutorials/source/srsUE/source/index.html#zeromq-based-setup)
**`srsRAN_Project/build/apps/gnb/gnb_zmq.yaml`**
```diff
+gnb_id: 0x19B
+
 cu_cp:
   amf:
-    addr: 10.53.1.2                 # The address or hostname of the AMF.
+    addr: 172.16.0.50                 # The address or hostname of the AMF.
     port: 38412
-    bind_addr: 10.53.1.1            # A local IP that the gNB binds to for traffic from the AMF.
+    bind_addr: 172.16.0.60            # A local IP that the gNB binds to for traffic from the AMF.
     supported_tracking_areas:
-      - tac: 7
+      - tac: 1
         plmn_list:
           - plmn: "00101"
             tai_slice_support_list:
               - sst: 1
+                sd: 1
   inactivity_timer: 7200            # Sets the UE/PDU Session/DRB inactivity timer to 7200 seconds. Supported: [1 - 7200].

+cu_up:
+  ngu:
+    socket:                         # Define socket(s) for NG-U interface.
+      - bind_addr: 172.16.0.60      # Optional TEXT (auto). Sets local IP address to bind for N3 interface. Format: IPV4 or IPV6 IP address.
+
 ru_sdr:
   device_driver: zmq                # The RF driver name.
-  device_args: tx_port=tcp://127.0.0.1:2000,rx_port=tcp://127.0.0.1:2001,base_srate=23.04e6 # Optionally pass arguments to the selected RF driver.
+  device_args: tx_port=tcp://172.16.0.60:2000,rx_port=tcp://172.16.0.61:2001,base_srate=23.04e6 # Optionally pass arguments to the selected RF driver.
   srate: 23.04                      # RF sample rate might need to be adjusted according to selected bandwidth.
   tx_gain: 75                       # Transmit gain of the RF might need to adjusted to the given situation.
   rx_gain: 75                       # Receive gain of the RF might need to adjusted to the given situation.
@@ -29,7 +35,7 @@
   channel_bandwidth_MHz: 20         # Bandwith in MHz. Number of PRBs will be automatically derived.
   common_scs: 15                    # Subcarrier spacing in kHz used for data.
   plmn: "00101"                     # PLMN broadcasted by the gNB.
-  tac: 7                            # Tracking area code (needs to match the core configuration).
+  tac: 1                            # Tracking area code (needs to match the core configuration).
   pdcch:
     common:
       ss0_index: 0                  # Set search space zero index to match srsUE capabilities
```
#### NR-UE
[Original file](https://docs.srsran.com/projects/project/en/latest/tutorials/source/srsUE/source/index.html#zeromq-based-setup)
**`srsRAN_4G/build/srsue/ue_zmq.conf`**
```diff
 nof_antennas = 1

 device_name = zmq
-device_args = tx_port=tcp://127.0.0.1:2001,rx_port=tcp://127.0.0.1:2000,base_srate=23.04e6
+device_args = tx_port=tcp://172.16.0.61:2001,rx_port=tcp://172.16.0.60:2000,base_srate=23.04e6

 [rat.eutra]
 dl_earfcn = 2850
@@ -34,9 +34,9 @@
 [usim]
 mode = soft
 algo = milenage
-opc  = 63BFA50EE6523365FF14C1F45F88737D
-k    = 00112233445566778899aabbccddeeff
-imsi = 001010123456780
+opc  = E8ED289DEBA952E4283B54E88E6183CA
+k    = 465B5CE8B199B49FAA5F0A2EE238A6BC
+imsi = 001010000000000
 imei = 353490069873319

 [rrc]
@@ -44,14 +44,18 @@
 ue_category = 4

 [nas]
-apn = srsapn
+apn = internet
 apn_protocol = ipv4

+#[slicing]
+#enable = true
+#nssai-sst = 1
+#nssai-sd = 1
+
 [gw]
-netns = ue1
+#netns = ue1
 ip_devname = tun_srsue
 ip_netmask = 255.255.255.0
```

---

## 4. Operation

### 4.1 Start 5GC C-Plane
Execute the following to start the Core Network services:
```bash
./install/bin/open5gs-nrfd &
sleep 2
./install/bin/open5gs-scpd &
sleep 2
./install/bin/open5gs-amfd &
sleep 2
./install/bin/open5gs-smfd &
./install/bin/open5gs-ausfd &
./install/bin/open5gs-udmd &
./install/bin/open5gs-udrd &
./install/bin/open5gs-pcfd &
./install/bin/open5gs-nssfd &
./install/bin/open5gs-bsfd &
```

### 4.2 Start 5GC U-Plane
```bash
./install/bin/open5gs-upfd &
```

### 4.3 Start gNodeB
```bash
cd srsRAN_Project/build/apps/gnb
./gnb -c gnb_zmq.yaml
```
*Expected Output:*
```text
--== srsRAN gNB (commit ABC) ==--

Lower PHY in executor blocking mode.
Available radio types: zmq.

_truncated_

N2: Connection to AMF on 172.16.0.50:38412 completed
==== gNB started ===
Type <h> to view help
```

### 4.4 Start UE
```bash
cd srsRAN_4G/build/srsue
./src/srsue ue_zmq.conf
```
*Expected Output:*
```text
Reading configuration file ue_zmq.conf...

Built in Release mode using commit XYZ on branch master.

Opening 1 channels in RF device=zmq with args=tx_port=tcp://172.16.0.61:2001,rx_port=tcp://172.16.0.60:2000,base_srate=23.04e6
Supported RF device list: zmq file
CHx base_srate=23.04e6
Current sample rate is 1.92 MHz with a base rate of 23.04 MHz (x12 decimation)
CH0 rx_port=tcp://172.16.0.60:2000
CH0 tx_port=tcp://172.16.0.61:2001
Current sample rate is 23.04 MHz with a base rate of 23.04 MHz (x1 decimation)
Current sample rate is 23.04 MHz with a base rate of 23.04 MHz (x1 decimation)
Waiting PHY to initialize ... done!
Attaching UE...

_truncated_

RRC Connected
PDU Session Establishment successful. IP: 10.45.0.2
RRC NR reconfiguration successful.
````

## 5. Known Issues

### AVX Instruction Support
Virtual Machines require access to AVX CPU instructions for successful compilation and execution. Without AVX support, you may encounter errors or compilation failures.

> **Proxmox:** Set **Processor Type** to `host` in VM hardware settings.

## 6. Acknowledgements

This guide is based on the excellent work by **s5uishida**.
- Original Guide: [build_srsran_5g_zmq](https://github.com/s5uishida/build_srsran_5g_zmq)
- Sample Configurations: [sample_config_misc_for_mobile_network](https://github.com/s5uishida/sample_config_misc_for_mobile_network)

While this document includes additional instructions for compiling binary files, the core configuration logic is derived from their repositories.
