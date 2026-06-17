# RF-free 5G SA baseline

The foundation of this study: srsRAN's 5G gNB attached to an **Open5GS** core
with a software UE (**srsUE**), over **ZMQ virtual RF** — no radio hardware.
Every efficiency number in this study is measured against this baseline, so it
must be reproducible by a teammate on a plain Linux box with no SDR.

## Host

Linux only — **does not build on macOS**. Verified on a UMN lab box: Ubuntu
20.04, x86-64, 24 cores, 251 GB RAM. A few GB of free disk is enough.

## Pinned versions

| Component | Version |
|---|---|
| srsRAN_Project (gNB) | `release_25_10` (commit `d2f4b70`) |
| srsRAN_4G (srsUE) | `release_25_10` |
| Open5GS (5G core) | 2.7.x |
| MongoDB (subscriber DB) | mongodb-org 8.0 |

## Ubuntu-20.04 build gotcha (read first)

`release_25_10` does **not** compile with the stock g++ 9.4: `-Werror` promotes
an attribute warning in a bundled dependency (uWebSockets) to a fatal error
(build dies ~127/1310). Fix — build with GCC ≥ 11 and relax `-Werror`:

```sh
sudo apt install -y g++-12 gcc-12
# then add to the gNB cmake line:
#   -DCMAKE_C_COMPILER=gcc-12 -DCMAKE_CXX_COMPILER=g++-12 -DENABLE_WERROR=OFF
```

On Ubuntu 22.04+ (stock GCC 11+) it builds directly.

## 1 · Core — MongoDB + Open5GS

```sh
# MongoDB: the universe `mongodb` package will NOT work — use mongodb-org.
curl -fsSL https://pgp.mongodb.com/server-8.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/8.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
sudo apt update && sudo apt install -y mongodb-org && sudo systemctl enable --now mongod

# Open5GS (apt PPA is simplest)
sudo add-apt-repository -y ppa:open5gs/latest && sudo apt install -y open5gs
```

## 2 · Data network (ogstun + NAT, so the UE reaches the Internet)

```sh
sudo ip tuntap add name ogstun mode tun
sudo ip addr add 10.45.0.1/16 dev ogstun
sudo ip link set ogstun up
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
```

## 3 · PLMN + subscriber

Set PLMN **001/01**, TAC **1**, slice **sst 1** in `/etc/open5gs/amf.yaml`, then
add the test subscriber (srsRAN/Open5GS well-known test credentials — not
secret):

```
IMSI  001010123456789
K     00112233445566778899AABBCCDDEEFF
OPc   63BFA50EE6523365FF14C1F45F88737D
APN   internet
```

Add it via the Open5GS WebUI (http://127.0.0.1:9999, admin/1423) or the
`open5gs-dbctl add <imsi> <k> <opc>` helper.

## 4 · Build gNB + srsUE

```sh
# gNB — ZMQ auto-detected when libzmq3-dev is present (verified on release_25_10)
git clone https://github.com/srsRAN/srsRAN_Project && cd srsRAN_Project
git checkout release_25_10 && mkdir build && cd build
cmake .. -G Ninja -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=gcc-12 -DCMAKE_CXX_COMPILER=g++-12 -DENABLE_WERROR=OFF
ninja                                   # -> apps/gnb/gnb

# srsUE
cd ~ && git clone https://github.com/srsRAN/srsRAN_4G && cd srsRAN_4G
git checkout release_25_10 && mkdir build && cd build
cmake .. -DCMAKE_C_COMPILER=gcc-12 -DCMAKE_CXX_COMPILER=g++-12 -DENABLE_WERROR=OFF
make -j"$(nproc)" srsue                 # -> srsue/src/srsue (RF plugin: libsrsran_rf_zmq.so)
```

## 5 · Configs (PLMN 001/01 · ZMQ 2000↔2001 · srate 23.04 MHz · band 3 / 106 PRB)

`gnb_zmq.yaml` (gNB tx = UE rx, gNB rx = UE tx):

```yaml
cu_cp:
  amf:
    addr: 127.0.0.5
    bind_addr: 127.0.0.1
    supported_tracking_areas:
      - tac: 1
        plmn_list:
          - plmn: "00101"
            tai_slice_support_list: [ { sst: 1 } ]
ru_sdr:
  device_driver: zmq
  device_args: tx_port=tcp://127.0.0.1:2000,rx_port=tcp://127.0.0.1:2001,base_srate=23.04e6
  srate: 23.04
  tx_gain: 75
  rx_gain: 75
cell_cfg:
  dl_arfcn: 368500          # band 3 FDD
  band: 3
  channel_bandwidth_MHz: 20 # 106 PRB
  common_scs: 15
  plmn: "00101"
  tac: 1
```

`ue.conf` (key fields):

```ini
[rf]
device_name=zmq
device_args=tx_port=tcp://127.0.0.1:2001,rx_port=tcp://127.0.0.1:2000,base_srate=23.04e6
srate=23.04e6
[rat.nr]
bands=3
nof_prb=106
[usim]
algo=milenage
imsi=001010123456789
k=00112233445566778899AABBCCDDEEFF
opc=63BFA50EE6523365FF14C1F45F88737D
[gw]
netns=ue1
```

## 6 · Run + verify

```sh
sudo ip netns add ue1
sudo systemctl start open5gs-*                 # 5G core NFs
sudo ./gnb -c gnb_zmq.yaml                      # terminal 2
sudo srsue ue.conf                              # terminal 3
sudo ip netns exec ue1 ping -c3 8.8.8.8         # verify
```

**Success = the UE attaches, gets a `10.45.0.0/16` address, and the ping
reaches `8.8.8.8`.** That round-trip over ZMQ *is* the baseline; throughput /
latency / spectral-efficiency / energy are then measured on top of it (e.g.
`iperf3` through the `ue1` namespace) and recorded under `../../runs/`.

## Status

Builds **verified on aum, 2026-06-17**: gNB `release_25_10` (g++-12, `-Werror`
off), srsUE `release_25_10`, Open5GS 2.7.7 + MongoDB. The end-to-end attach is
being re-verified on these exact pinned builds; the run/config values above are
from a previously working single-host setup.

## Troubleshooting

- **UE won't attach** → PLMN/TAC mismatch, or ZMQ ports swapped (gNB `tx_port`
  must equal UE `rx_port`, and vice-versa), or `base_srate` differs between the two.
- **Attaches but no Internet** → `ogstun` down, `net.ipv4.ip_forward` off, or the
  MASQUERADE rule missing.
- **gNB build fails on Ubuntu 20.04** → use g++-12 + `-DENABLE_WERROR=OFF` (above).
