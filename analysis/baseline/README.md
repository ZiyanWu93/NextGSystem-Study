# RF-free 5G-SA baseline — VERIFIED working

srsRAN's 5G gNB attached to an **Open5GS** core with a software UE (**srsUE**)
over **ZMQ virtual RF** — no radio hardware. Every efficiency number in this
study is measured against this baseline.

> **Status: VERIFIED end-to-end on 2026-06-17.** The UE attaches, gets
> `10.45.0.2`, and `ping 8.8.8.8` from the UE returns **0% packet loss**
> (RTT ~30–170 ms over ZMQ). Configs below are the exact tested ones.

## Host

Linux only (does **not** build on macOS). Verified on a UMN box: Ubuntu 20.04,
x86-64, 24 cores, 251 GB RAM.

## Pinned versions

| Component | Version |
|---|---|
| srsRAN_Project (gNB) | `release_25_10` (commit `d2f4b70`) |
| srsRAN_4G (srsUE) | `release_25_10` (commit `6bcbd9e`) |
| Open5GS (5G core) | `2.7.7~focal` (apt PPA) |
| MongoDB | mongodb-org **7.0** (NOT the universe `mongodb` pkg) |
| Compiler | `g++-12` / `gcc-12` (Ubuntu 20.04 ships gcc 9.4 — too old) |
| ZMQ | `libzmq` + `czmq` built from source into `/usr/local` |

## The fixes that actually mattered (hard-won — read before reproducing)

1. **gNB `pdcch.common.coreset0_index: 6`** for band 3 / 10 MHz / SCS 15. A
   minimal config (omitting pdcch, or a wrong index) leaves CORESET#0 mapped
   outside the carrier → **SIB1 undecodable → RRC "Cell Selection" fails → no
   RACH**. This was the single biggest blocker. (3GPP 38.213 Table 13-1;
   srsRAN issue #1278. For 20 MHz the matched value is `coreset0_index: 12`.)
2. **gNB `pdcch.dedicated.dci_format_0_1_and_1_1: false`** — srsUE only handles
   fallback DCI; without this the UE won't be scheduled.
3. **One coherent bandwidth profile on BOTH ends**: 10 MHz → 52 PRB → SCS 15 →
   `srate`/`base_srate` **11.52e6** on gNB *and* UE. Mismatched srate = silent
   sync failure.
4. **ZMQ `id=` + bind/connect form**: gNB `tx=tcp://127.0.0.1:2000,id=gnb`,
   UE `tx=tcp://*:2001,rx=tcp://localhost:2000,id=ue`. **Co-restart gNB+UE
   together** — restarting only one leaves the ZMQ REQ/REP socket stale.
5. **PLMN 001/01 consistent across ALL Open5GS NF configs** (`sed -i
   's/mcc: 999/mcc: 001/g; s/mnc: 70/mnc: 01/g' /etc/open5gs/*.yaml`), not just
   `amf.yaml`. Otherwise the SCP NF-discovery fails (`No SEPP / NF-Discover
   504`) → AMF `HTTP 400` → **Registration reject [95]**.
6. **Subscriber needs `security.sqn`**: `db.subscribers.updateOne({imsi:...},
   {$set:{"security.sqn":NumberLong(0)}})` — some `dbctl` versions omit it.
7. **Default route in the UE namespace**: `ip netns exec ue1 ip route replace
   default dev tun_srsue` — srsUE assigns the IP but not the route.
8. **Ubuntu-20.04 build**: srsRAN `release_25_10` needs GCC ≥ 11 — build with
   `g++-12` and `-DENABLE_WERROR=OFF` (stock g++ 9.4 dies on a uWebSockets
   `-Werror` warning). On Ubuntu 22.04+ this is moot.

## 0 · Toolchain, ZMQ, and build

```sh
# Host deps (Ubuntu 20.04/22.04, x86-64). 20.04 ships gcc 9.4 → add g++-12.
sudo apt update
sudo apt install -y cmake make gcc g++ pkg-config build-essential libtool \
  autoconf automake libfftw3-dev libmbedtls-dev libsctp-dev libyaml-cpp-dev \
  libgtest-dev libuhd-dev uhd-host libboost-program-options-dev libconfig++-dev
sudo apt install -y gcc-12 g++-12        # Ubuntu 20.04 only

# ZMQ virtual radio: libzmq + czmq from source (the srsRAN-documented path; the
# RF driver links against /usr/local/lib/libzmq.so). apt's libzmq3-dev also works.
mkdir -p ~/src && cd ~/src
git clone https://github.com/zeromq/libzmq.git
cd libzmq && ./autogen.sh && ./configure && make -j"$(nproc)" && sudo make install && sudo ldconfig && cd ..
git clone https://github.com/zeromq/czmq.git
cd czmq  && ./autogen.sh && ./configure && make -j"$(nproc)" && sudo make install && sudo ldconfig && cd ..

# gNB — srsRAN_Project. g++-12 + WERROR=OFF (fix #8); ZMQ auto-detected from libzmq.
git clone https://github.com/srsran/srsRAN_Project.git ~/srsRAN_Project
cd ~/srsRAN_Project && git checkout release_25_10
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release -DENABLE_WERROR=OFF \
      -DCMAKE_C_COMPILER=gcc-12 -DCMAKE_CXX_COMPILER=g++-12 ..
make -j"$(nproc)" gnb        # -> build/apps/gnb/gnb

# srsUE — srsRAN_4G.
git clone https://github.com/srsran/srsRAN_4G.git ~/srsRAN_4G
cd ~/srsRAN_4G && git checkout release_25_10
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release -DENABLE_WERROR=OFF \
      -DCMAKE_C_COMPILER=gcc-12 -DCMAKE_CXX_COMPILER=g++-12 ..
make -j"$(nproc)" srsue      # -> build/srsue/src/srsue
```

## 1 · Core (MongoDB + Open5GS)

```sh
# MongoDB 7.0 (the universe `mongodb` 3.6 pkg does NOT work with Open5GS 2.7)
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg] https://repo.mongodb.org/apt/ubuntu $(lsb_release -cs)/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
sudo apt update && sudo apt install -y mongodb-org && sudo systemctl enable --now mongod

sudo add-apt-repository -y ppa:open5gs/latest && sudo apt install -y open5gs

# PLMN 001/01 across ALL NFs (fix #5) + slice sst 1 in amf.yaml
sudo sed -i 's/mcc: 999/mcc: 001/g; s/mnc: 70/mnc: 01/g' /etc/open5gs/*.yaml

# data network
sudo ip tuntap add name ogstun mode tun; sudo ip addr add 10.45.0.1/16 dev ogstun; sudo ip link set ogstun up
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE

# subscriber (test creds) + sqn (fix #6); then restart NFs NRF-first
open5gs-dbctl add 001010123456789 00112233445566778899AABBCCDDEEFF 63BFA50EE6523365FF14C1F45F88737D
mongosh open5gs --eval 'db.subscribers.updateOne({imsi:"001010123456789"},{$set:{"security.sqn":NumberLong("0")}})'
sudo systemctl restart open5gs-nrfd; sleep 3; sudo systemctl restart open5gs-scpd open5gs-udrd open5gs-udmd open5gs-ausfd open5gs-pcfd open5gs-bsfd open5gs-nssfd open5gs-smfd open5gs-upfd open5gs-amfd
```

## 2 · gNB config — `gnb_zmq.yaml` (verified)

```yaml
cu_cp:
  amf:
    addr: 127.0.0.5
    port: 38412
    bind_addr: 127.0.0.1
    supported_tracking_areas:
      - tac: 1
        plmn_list:
          - plmn: "00101"
            tai_slice_support_list:
              - sst: 1
ru_sdr:
  device_driver: zmq
  device_args: tx_port=tcp://127.0.0.1:2000,rx_port=tcp://127.0.0.1:2001,id=gnb,base_srate=11.52e6
  srate: 11.52          # MHz (bare number); device_args base_srate is in Hz — both = 11.52 MHz
  tx_gain: 75
  rx_gain: 75
cell_cfg:
  dl_arfcn: 368500
  band: 3
  channel_bandwidth_MHz: 10
  common_scs: 15
  plmn: "00101"
  tac: 1
  pci: 1
  pdcch:
    common: { ss0_index: 0, coreset0_index: 6 }        # fix #1
    dedicated: { ss2_type: common, dci_format_0_1_and_1_1: false }  # fix #2
  prach: { prach_config_index: 1 }
  pdsch: { mcs_table: qam64 }
  pusch: { mcs_table: qam64 }
log: { filename: /tmp/gnb.log, all_level: info }
```

## 3 · UE config — `ue_zmq.conf` (verified)

```ini
[rf]
freq_offset = 0
tx_gain = 80
srate = 11.52e6
device_name = zmq
device_args = tx_port=tcp://*:2001,rx_port=tcp://localhost:2000,id=ue,base_srate=11.52e6
[rat.eutra]
nof_carriers = 0          # disable LTE -> pure 5G SA
[rat.nr]
bands = 3
nof_carriers = 1
max_nof_prb = 52
nof_prb = 52
[usim]
mode = soft
algo = milenage
opc  = 63BFA50EE6523365FF14C1F45F88737D
k    = 00112233445566778899AABBCCDDEEFF
imsi = 001010123456789
[rrc]
release = 15
[nas]
apn = internet
[gw]
netns = ue1
[log]
all_level = info
filename = /tmp/ue.log
```

## 4 · Run + verify

```sh
sudo ip netns add ue1
sudo ./gnb -c gnb_zmq.yaml                 # wait for "Cell ... activated" + "Connected to AMF"
sudo ./srsue ue_zmq.conf                    # co-restart with the gNB (fix #4)
# UE log should show: RRC Connected -> Registration Accept -> PDU Session Establishment successful. IP: 10.45.0.2
sudo ip netns exec ue1 ip route replace default dev tun_srsue   # fix #7
sudo ip netns exec ue1 ping -c4 8.8.8.8     # SUCCESS = 0% loss
```

**Verified result:** `tun_srsue 10.45.0.2/24`; `ping 8.8.8.8 → 4/4 received, 0% loss`.
That round-trip over ZMQ is the baseline; throughput / latency / spectral
efficiency / energy are measured on top of it (e.g. `iperf3` through `ue1`) and
recorded under `../../runs/`.
