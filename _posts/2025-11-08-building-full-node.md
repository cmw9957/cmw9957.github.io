---
title: 'Ethereum Full-node 구축하기'
date: 2025-11-08
permalink: /posts/building-full-node
tags:
  - geth
  - lighthouse
  - full node
---

약 한달 전에 arbitrage bot 구현을 위해 full node를 구축했습니다. full node 운영 이유는 빠르게 on-chain 데이터를 읽어오기 위함이고, 제한이 없기 때문입니다. 또한, node에서 사용할 수있는 여러 tracer들을 온전히 사용할 수 있고, 목적에 따라 커스텀하기 위해 운영하기로 마음 먹었습니다.

# Hardware requirement
<img src="../images/2025-11-08/hardware-requirement.png" alt="alt text" width="500">

- CPU : 14 cores / 20 threads(6 performance core, 8 efficient core)
- RAM : 32GB
- Storage : 4TB NVMe SSD

총 1,545,290원의 비용이 들었습니다. 부품 관련 문의는 직접 Ethereum 2.0 Staking User Community 네이버 카페에 [질문글](https://cafe.naver.com/eth2staking?iframe_url_utf8=%2FArticleRead.nhn%253Fclubid%3D30197760%2526articleid%3D873)을 올렸습니다.

# Execution & Consensus client
## geth
[geth](https://github.com/ethereum/go-ethereum)는 Ethereum protocol의 Go 언어로 이루어진 execution layer 구현체입니다. geth는 다음과 같은 단계로 transaction을 처리합니다.

### 1. transaction 수신 및 검증
RPC API 또는 P2P 네트워크를 통해 transaction을 수신합니다. 서명 검증(ECDSA), nonce 확인, 계정 잔액 검증 등 기본적인 유효성 검사를 수행합니다. 검증에 실패하면 즉시 에러를 반환하고, 성공하면 transaction hash를 반환합니다.

### 2. Mempool 추가 및 네트워크 전파
검증을 통과한 transaction은 mempool에 pending 상태로 추가됩니다. DevP2P 프로토콜을 사용하여 연결된 다른 Ethereum 노드들에게 transaction을 전파합니다. Gossip 방식으로 몇 초 내에 전체 네트워크로 확산됩니다.

### 3. transaction 실행 및 상태 관리
Validator 노드가 블록을 생성할 차례가 되면, mempool에서 가스비가 높은 순서대로 transaction을 선택합니다. block gas limit 내에서 transaction들을 EVM으로 실행하고, 실행 결과에 따라 계정의 balance, nonce, storage 등을 업데이트합니다. Merkle Patricia Trie로 새로운 state root를 계산하여 execution payload를 생성합니다.

### 4. 블록 검증
다른 노드가 제안한 블록을 받으면, 블록 내 모든 transaction을 순서대로 재실행합니다. 계산한 state root가 블록에 명시된 값과 일치하는지 확인하고, 일치하면 블록을 로컬 체인에 추가합니다. 이 시점에서 transaction이 확정되고 mempool에서 제거됩니다.


## lighthouse
[lighthouse](https://github.com/sigp/lighthouse)는 Ethereum protocol의 Rust 언어로 이루어진 consensus layer 구현체입니다. Lighthouse는 다음과 같은 단계로 합의를 이루고 블록을 최종 확정합니다.

### 1. 블록 제안 (Block Proposal)
Validator가 자신의 차례가 되면 geth에게 execution payload 생성을 요청합니다(`engine_forkchoiceUpdated`). Geth로부터 받은 payload에 attestation, 슬래싱 증거 등을 포함하여 완전한 beacon block을 구성합니다. BLS 서명으로 블록에 서명한 후 P2P 네트워크를 통해 다른 노드들에게 전파합니다.

### 2. Attestation 및 검증 (Attestation & Validation)
각 슬롯마다 validator는 현재 체인의 헤드 블록에 대한 attestation(투표)을 수행합니다. 다른 노드로부터 블록을 받으면 beacon chain 규칙(슬롯, 서명 등)을 확인하고, geth에게 execution payload 검증을 요청합니다(`engine_newPayload`). 검증이 완료되면 LMD-GHOST 알고리즘을 사용하여 가장 많은 attestation을 받은 체인을 정규 체인으로 선택합니다. Aggregator 역할을 맡은 validator는 같은 위원회의 attestation들을 BLS 서명으로 병합하여 블록 제안자에게 전달합니다.

### 3. 최종성 보장 (Finality)
Casper FFG 메커니즘을 통해 블록의 최종성을 보장합니다. Checkpoint(epoch의 첫 슬롯)에 대해 2/3 이상의 validator가 투표하면 justified 상태가 되고, 연속된 두 justified checkpoint가 생기면 이전 checkpoint가 finalized 됩니다. Finalized된 블록은 절대 되돌릴 수 없는 경제적 최종성을 가지며, 보통 2 epoch(약 12.8분) 후에 달성됩니다. Double proposal이나 surround vote 같은 악의적 행동을 감지하면 슬래싱 증거를 블록에 포함시켜 해당 validator를 처벌합니다.

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

# reference
- <https://medium.a41.io/%EC%9D%B4%EB%8D%94%EB%A6%AC%EC%9B%80-%EC%86%94%EB%A1%9C-validator-%EB%85%B8%EB%93%9C-%EA%B5%AC%EC%B6%95%ED%95%98%EA%B8%B0-af2013e62ef7>

- <https://cafe.naver.com/eth2staking?iframe_url_utf8=%2FArticleRead.nhn%253Fclubid%3D30197760%2526articleid%3D873>