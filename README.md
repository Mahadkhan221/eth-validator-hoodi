# Ethereum Validator Setup (Hoodi Testnet)
## Geth + Lighthouse + Prometheus + Grafana (systemd)

This repository documents my complete Ethereum **Hoodi testnet validator** setup.
All services are managed using **systemd** and can be reproduced step by step.

---

## Components Used

- Geth (Execution Client)
- Lighthouse Beacon Node (Consensus Client)
- Lighthouse Validator Client
- Prometheus (Metrics collection)
- Grafana (Monitoring dashboards)
- Optional: ngrok for temporary Grafana access

---

## Repository Structure

.
├── geth/
│ └── systemd/
│ └── geth.service
│
├── lighthouse/
│ ├── systemd/
│ │ ├── lighthouse-beacon.service
│ │ └── lighthouse-validator.service
│ └── validator/
│ └── validator_definitions.yml
│
├── prometheus/
│ └── config/
│ └── prometheus.yml
│
├── grafana/
│ └── dashboards/
│ └── hoodi-dashboard.json
│
└── README.md

yaml


---

## System Requirements

- Ubuntu 22.04 or newer
- 4+ CPU cores
- 8 GB RAM or more
- SSD storage
- Stable internet connection

---
### Grafana
A ready-to-use dashboard is included in:
grafana/dashboards/hoodi-dashboard.json

You can import it via:
Grafana → Dashboards → Import → Upload JSON

## Base System Setup

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget jq git ufw vim

Create a dedicated Ethereum user and data directory:

bash

sudo useradd -r -s /bin/false ethereum
sudo mkdir -p /data
sudo chown -R ethereum:ethereum /data

Geth (Execution Client)

Install Geth

bash

sudo add-apt-repository -y ppa:ethereum/ethereum
sudo apt update
sudo apt install -y geth

Create JWT Secret
bash

openssl rand -hex 32 | sudo tee /data/jwt.hex
sudo chown ethereum:ethereum /data/jwt.hex
sudo chmod 600 /data/jwt.hex

Enable Geth systemd Service
bash

sudo cp geth/systemd/geth.service /etc/systemd/system/geth.service
sudo systemctl daemon-reload
sudo systemctl enable --now geth

Verify:

bash
sudo systemctl status geth --no-pager

Lighthouse Beacon Node

Install Lighthouse
bash

curl -LO https://github.com/sigp/lighthouse/releases/download/v8.0.1/lighthouse-v8.0.1-x86_64-unknown-linux-gnu.tar.gz
tar -xvf lighthouse-v8.0.1-x86_64-unknown-linux-gnu.tar.gz
sudo mv lighthouse /usr/local/bin/

Create Lighthouse Directories
bash

sudo mkdir -p /data/lighthouse/{beacon,validator}
sudo chown -R ethereum:ethereum /data/lighthouse

Enable Beacon Node Service
bash

sudo cp lighthouse/systemd/lighthouse-beacon.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now lighthouse-beacon

Check sync status:

bash

curl -s http://127.0.0.1:5052/eth/v1/node/syncing ; echo

While syncing, warnings about the beacon being behind are normal.

Validator Client

Validator Keys

Validator keys are generated using the official Ethereum staking-deposit-cli.
Offline key generation is strongly recommended.

Keystore files must be placed under:

swift

/data/lighthouse/validator/validators/

Validator Definitions
Edit the file:

bash

sudo -u ethereum vim /data/lighthouse/validator/validator_definitions.yml

Example:

yaml

- enabled: true
  voting_public_key: 0xYOUR_VALIDATOR_PUBLIC_KEY
  type: local_keystore
  voting_keystore_path: /data/lighthouse/validator/validators/0x.../keystore.json
  voting_keystore_password: YOUR_KEYSTORE_PASSWORD

Enable Validator Service
bash

sudo cp lighthouse/systemd/lighthouse-validator.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now lighthouse-validator

Check logs:

bash

sudo journalctl -u lighthouse-validator -f

Prometheus

Install and Configure

bash

sudo apt install -y prometheus
sudo cp prometheus/config/prometheus.yml /etc/prometheus/prometheus.yml
sudo systemctl restart prometheus

Verify scrape targets:

bash

curl -s "http://localhost:9090/api/v1/query?query=up" | jq

**Grafana**

Install Grafana

bash

sudo apt install -y grafana
sudo systemctl enable --now grafana-server
Access Grafana:

cpp

http://SERVER_IP:3000

Default login:

Username: admin

Password: admin

Import Dashboard
Open Grafana

Go to Dashboards → Import

Upload grafana/dashboards/hoodi-dashboard.json

Select Prometheus as the data source

Optional: Grafana Access via ngrok

If SSH port forwarding is restricted, Grafana can be exposed temporarily using ngrok:

bash

ngrok http 3000
This creates a temporary HTTPS URL.
The tunnel stops when the SSH session closes and does not affect the validator.

Demo / Verification Commands

Show running services:

bash

sudo systemctl status geth lighthouse-beacon lighthouse-validator prometheus grafana-server --no-pager

Live validator logs:

bash

sudo journalctl -u lighthouse-validator -f

Beacon sync status:

bash

curl -s http://127.0.0.1:5052/eth/v1/node/syncing ; echo

Prometheus health check:

bash

curl -s "http://localhost:9090/api/v1/query?query=up" | jq


Notes:

Beacon sync warnings are expected until fully synced

Validator metrics may show zero until duties begin

Grafana access is optional and does not affect node operation

All services continue running after SSH disconnect

This README fully reproduces my Ethereum Hoodi testnet validator setup.
