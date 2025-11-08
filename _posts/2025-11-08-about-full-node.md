---
title: 'Setting up a full node'
date: 2025-11-08
permalink: /posts/about-full-node
tags:
  - geth
  - lighthouse
  - full node
---

약 한달 전에 arbitrage bot 구현을 위해 full node를 구축했습니다. full node 운영 이유는 빠르게 on-chain 데이터를 읽어오기 위함이고, 제한이 없기 때문입니다. 또한, node에서 사용할 수있는 여러 tracer들을 온전히 사용할 수 있고, 목적에 따라 커스텀하기 위해 운영하기로 마음 먹었습니다.

# Hardware requirement
<img src="../images/hardware-requirement.png" alt="alt text" width="500">

- CPU : 14 cores / 20 threads(6 performance core, 8 efficient core)
- RAM : 32GB
- Storage : 4TB NVMe SSD

총 1,545,290원의 비용이 들었습니다.

# Execution & Consensus client
## geth
## lighthouse

# geth install

**기본 업데이트**
``` bash
sudo apt update -y && sudo apt upgrade -y && sudo apt dist-upgrade -y && sudo apt autoremove -y
```

**시간 설정**
``` bash
sudo timedatectl set-timezone Asia/Seoul

timedatectl status
```

**jwt 생성**
``` bash
cd ~

sudo mkdir -p /var/lib/jwtsecret

openssl rand -hex 32 | sudo tee /var/lib/jwtsecret/jwt.hex > /dev/null
```

**geth 설치**
``` bash
sudo add-apt-repository -y ppa:ethereum/ethereum

sudo apt-get install ethereum -y
```

**geth 관련 설정**
``` bash
sudo useradd --no-create-home --shell /bin/false geth

sudo mkdir -p /var/lib/geth

sudo chown -R geth:geth /var/lib/geth
```

**geth 서비스 파일 생성**
``` bash
sudo vim /etc/systemd/system/geth.service
```

```
[Unit]
Description=Geth
After=network-online.target
[Service]
Type=simple
User=geth
ExecStart=geth \
  --datadir /var/lib/geth \
  --metrics \
  --pprof \
  --http \
  --http.addr=0.0.0.0 \
  --port=20001 \
  --authrpc.jwtsecret /var/lib/jwtsecret/jwt.hex \
  --authrpc.vhosts="*" \
  --state.scheme=path
Restart=always
RestartSec=5
TimeoutSec=900
[Install]
WantedBy=multi-user.target
```

# lighthouse install

**lighthouse binary 압축 해제**
``` bash
curl -LO https://github.com/sigp/lighthouse/releases/download/v8.0.0/lighthouse-v8.0.0-x86_64-unknown-linux-gnu.tar.gz

tar xvf lighthouse-v8.0.0-x86_64-unknown-linux-gnu.tar.gz

sudo cp lighthouse /usr/local/bin
```

**lighthouse 계정 생성**
``` bash
sudo useradd --no-create-home --shell /bin/false lighthousebeacon
```

**lighthouse 데이터 파일 생성 및 권한 설정**
``` bash
sudo mkdir -p /var/lib/lighthouse/beacon

sudo chown -R lighthousebeacon:lighthousebeacon /var/lib/lighthouse/beacon

sudo chmod 700 /var/lib/lighthouse/beacon
```

**lighthouse 서비스 파일 생성**
``` bash
vim /etc/systemd/system/lighthousebeacon.service
```

```
[Unit]
Description=Lighthouse Beacon Node
After=geth.target

[Service]
Type=simple
User=lighthousebeacon
Group=lighthousebeacon

ExecStart=/usr/local/bin/lighthouse bn \
  --network mainnet \
  --datadir /var/lib/lighthouse \
  --http \
  --http-address=0.0.0.0 \
  --validator-monitor-auto \
  --metrics \
  --execution-endpoint http://localhost:8551 \
  --execution-jwt /var/lib/jwtsecret/jwt.hex \
  --port 9002 \
  --discovery-port 9002 \
  --checkpoint-sync-url https://beaconstate.ethstaker.cc \
  --builder http://localhost:18550

Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fcmw9957%2FUpside_Practice_ERC20%2Fblob%2F6ddb2d063705a0066d062ad61c173f5281082117%2Fscript%2FCounter.s.sol%23L1-L12&style=vs2015&type=code&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

# reference
- <https://medium.a41.io/%EC%9D%B4%EB%8D%94%EB%A6%AC%EC%9B%80-%EC%86%94%EB%A1%9C-validator-%EB%85%B8%EB%93%9C-%EA%B5%AC%EC%B6%95%ED%95%98%EA%B8%B0-af2013e62ef7>

- <https://cafe.naver.com/eth2staking?iframe_url_utf8=%2FArticleRead.nhn%253Fclubid%3D30197760%2526articleid%3D873>