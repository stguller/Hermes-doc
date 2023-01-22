# Qwoyn <> Osmosis Hermes relayer setup
This document is intended only for setting up Hermes IBC relay and supporting the IBC channels announced by the Cosmic Horizon team. 
It does not contain any code to create channels on Hermes IBC. 

## Assumptions

In this document, we assume that you are already running an Osmosis and Qwoyn mainnet node. If you don't have any of them, you can try to install Hermes IBC by finding public RPC and port. The Osmosis-Qwoyn node will be able to communicate via the Hermes node. In the document, Osmosis and Qwoyn node ports are prepared using standard communication ports.

In these instructions, Hermes is installed under /srv/hermes, adjust the paths according to your setup.

These instructions are based on installation on recent Ubuntu.

## Hermes Official Documentation https://hermes.informal.systems/

Pre-requisites:

Add system user for hermes
```
adduser hermes
usermod -aG sudo hermes
```
```
sudo apt update && sudo apt upgrade -y
sudo apt install librust-openssl-dev build-essential git -y
```
Install rust (https://www.rust-lang.org/tools/install)
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

## Hermes Installation

```
git clone https://github.com/informalsystems/ibc-rs.git hermes
cd hermes
git checkout v1.2.0   ### Can check latest version https://hermes.informal.systems/installation.html#install-by-downloading
cargo build --release
sudo cp target/release/hermes /usr/bin
```

## Hermes Configuration

Make hermes config directory:

```
mkdir -p $HOME/.hermes
```

Create default hermes configuration, in this example are created IBC relayer between qwoyn <-> osmosis. 

Dont forget to change ip address to your rpc service if its not hosted on local vps and "memo_prefix = '<Service provider> IBC service'"

```
cat <<EOF > /$HOME/.hermes/config.toml
[global]
log_level = 'debug'

[mode]

[mode.clients]
enabled = false
refresh = false
misbehaviour = false

[mode.connections]
enabled = false

[mode.channels]
enabled = false

[mode.packets]
enabled = true
clear_interval = 100
clear_on_start = true
#filter = enabled
tx_confirmation = false

[rest]
enabled = true
host = '127.0.0.1'
port = 3000

[telemetry]
enabled = false
host = '127.0.0.1'
port = 3001

[[chains]]
id = 'osmosis-1'
rpc_addr = 'http://localhost:26657'
websocket_addr = 'ws://localhost:26657/websocket'
grpc_addr = 'http://localhost:9090'
rpc_timeout = '30s'
account_prefix = 'osmo'
key_name = 'relayer'
store_prefix = 'ibc'
max_gas = 15000000
max_msg_num = 10
gas_price = { price = 0.0001, denom = 'uosmo' }
gas_adjustment = 1
clock_drift = '15s'
trusting_period = '9days'
trust_threshold = { numerator = '1', denominator = '3' }
memo_prefix = '<Service provider> IBC service'
[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-XXXX']
]

[[chains]]
id = 'qwoyn-1'
rpc_addr = 'http://localhost:26657'
grpc_addr = 'http://localhost:9090'
websocket_addr = 'ws://localhost:26657/websocket'
rpc_timeout = '10s'
account_prefix = 'qwoyn'
key_name = 'relayer'
store_prefix = 'ibc'
max_gas = 2000000
gas_price = { price = 0.0025, denom = 'uqwoyn' }
gas_adjustment = 0.1
clock_drift = '300s'
trusting_period = '10days'
trust_threshold = { numerator = '1', denominator = '3' }
memo_prefix = '<Service provider> IBC service'
[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-XXX']
  ]
EOF

```
You can validate your hermes configuration file:
```
hermes config validate
INFO ThreadId(01) using default configuration from '/home/relay/.hermes/config.toml'
Success: "validation passed successfully"
```

Add your relaying-wallets to hermes' keyring

Best practice is to use the same mnemonic over all networks, do not use your relaying-addresses for anything else because it might lead to mismatched account sequence errors.
```
hermes keys restore osmosis-1 -m "12 or 24 magic words"
hermes keys restore juno-1 -m "12 or 24 magic words"
hermes keys restore stargaze-1 -m "12 or 24 magic words"
```

## Final steps
Create daemon service file

```
sudo tee /etc/systemd/system/hermes.service > /dev/null <<EOF
[Unit]
  Description=Hermes relayer daemon
  After=network-online.target
[Service]
  User=$USER
  ExecStart=/usr/bin/hermes start
  Restart=on-failure
  RestartSec=3
  LimitNOFILE=4096
[Install]
  WantedBy=multi-user.target
EOF
```
Start hermes service 

```
sudo systemctl enable hermes
sudo systemctl daemon-reload
sudo systemctl restart hermes && journalctl -u hermes.service -f
```
