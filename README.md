# Nolus Testnet Installation With Snapshot
Chain ID: nolus-rila | Latest Version Tag: v0.1.43 | Custom Port: 43

## Install Dependencies

```
sudo apt update
sudo apt install curl git jq lz4 build-essential
sudo apt upgrade
```
### Install Golang

```
sudo rm -rf /usr/local/go
curl -Ls https://go.dev/dl/go1.19.6.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
eval $(echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh)
eval $(echo 'export PATH=$PATH:$HOME/go/bin' | tee -a $HOME/.profile)
```

## Download Binaries and Build
### Clone Project Repository

```
git clone https://github.com/Nolus-Protocol/nolus-core.git
cd nolus-core
git checkout v0.1.43
```

### Build Project

```
make build
```
## Install Cosmovisor and create a service
### Prepare binaries for Cosmovisor

```
mkdir -p $HOME/.nolus/cosmovisor/genesis/bin
mv target/release/nolusd $HOME/.nolus/cosmovisor/genesis/bin/
rm -rf build
```
### Create symlinks to cosmosvisor

```
ln -s $HOME/.nolus/cosmovisor/genesis $HOME/.nolus/cosmovisor/current
sudo ln -s $HOME/.nolus/cosmovisor/current/bin/nolusd /usr/local/bin/nolusd
```

### Download and install Cosmovisor
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.4.0
```
### Create Nolus service

```
sudo tee /etc/systemd/system/nolusd.service > /dev/null << EOF
[Unit]
Description=nolus-testnet node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.nolus"
Environment="DAEMON_NAME=nolusd"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.nolus/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF
```

## Initilize the node

### Set node configuration
```
nolusd config chain-id nolus-rila
nolusd config keyring-backend test
nolusd config node tcp://localhost:43657
MONIKER="MONIKER_NAME"
nolusd init $MONIKER --chain-id nolus-rila
```
### Download genesis
```
wget -O $HOME/.nolus/config/genesis.json "https://raw.githubusercontent.com/Nolus-Protocol/nolus-networks/main/testnet/nolus-rila/genesis.json"
```
### Add seeds
```
sed -i -e "s|^seeds *=.*|seeds = \"3f472746f46493309650e5a033076689996c8881@nolus-testnet.rpc.kjnodes.com:43659\"|" $HOME/.nolus/config/config.toml
```
### Set minimum gas price and pruning
```
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.0025unls\"|" $HOME/.nolus/config/app.toml
sed -i \
-e 's|^pruning *=.*|pruning = "nothing"|' \ 
$HOME/.nolus/config/app.toml
```
### Set custom ports
```
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:43658\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:43657\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:43060\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:43656\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":43660\"%" $HOME/.nolus/config/config.toml
sed -i -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:43317\"%; s%^address = \":8080\"%address = \":43080\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:43090\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:43091\"%; s%^address = \"0.0.0.0:8545\"%address = \"0.0.0.0:43545\"%; s%^ws-address = \"0.0.0.0:8546\"%ws-address = \"0.0.0.0:43546\"%" $HOME/.nolus/config/app.toml
```

## Download latest nolus chain snapshot
```
curl -L https://snapshots.togss.me/nolus-snapshot/nolus-latest-snapshot.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.nolus
[[ -f $HOME/.nolus/data/upgrade-info.json ]] && cp $HOME/.nolus/data/upgrade-info.json $HOME/.nolus/cosmovisor/genesis/upgrade-info.json
```

## Start the node and check the logs
```
sudo systemctl daemon-reload
sudo systemctl enable nolusd
sudo systemctl start nolusd && sudo journalctl -u nolusd -f --no-hostname -o cat
```









































