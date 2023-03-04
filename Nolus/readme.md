![nolus](https://github.com/Lefey/Lefey/raw/main/img/nolus.png) 
# Nolus Rila Testnet guide

# Links
Offical website: [https://nolus.io](https://nolus.io)\
Docs: [https://docs-nolus-protocol.notion.site/Nolus-Protocol-Docs-a0ddfe091cc5456183417a68502705f8](https://docs-nolus-protocol.notion.site/Nolus-Protocol-Docs-a0ddfe091cc5456183417a68502705f8)\
GitHub: [https://github.com/Nolus-Protocol/nolus-core](https://github.com/Nolus-Protocol/nolus-core)\
Explorer: [https://nolus.explorers.guru/](https://nolus.explorers.guru)

# 1.Install dependencies

## Basic utils
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl git jq lz4 build-essential
```
## GO 1.19.6
```bash
sudo rm -rf /usr/local/go
curl -sL https://go.dev/dl/go1.19.6.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile 
source $HOME/.bash_profile
go version
```
# 2.Build node binary
```bash
cd && git clone https://github.com/Nolus-Protocol/nolus-core
cd nolus-core
git checkout v0.1.43
make install
```
```bash
nolusd version --long
```
> version: 0.1.43\
> commit: 90c21c44b5a4cae64e192ae2cabdeec9edec4736`

# 3.Configure node
## Initial config
```bash
nolusd init <moniker_name> --chain-id nolus-rila
nolusd config chain-id nolus-rila
nolusd config keyring-backend test
```
## Create/recover wallet
```bash
nolusd keys add <wallet_name>
nolusd keys add <wallet_name> --recover
```
## Download Genesis
```bash
wget -O $HOME/.nolus/config/genesis.json "https://raw.githubusercontent.com/Nolus-Protocol/nolus-networks/main/testnet/nolus-rila/genesis.json"
```
## Check Genesis
```bash
sha256sum $HOME/.nolus/config/genesis.json
```
> d22ea6488afe58478c54afeb2d6b5a45622c797dfd75c91a8653eb1f094173c5

## Minimum gas price/Peers/Filter peers/Max peers
```bash
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.0025unls\"/" $HOME/.nolus/config/app.toml
sed -i -e "s/^filter_peers *=.*/filter_peers = \"true\"/" $HOME/.nolus/config/config.toml
peers="$(curl -s "https://raw.githubusercontent.com/Nolus-Protocol/nolus-networks/main/testnet/nolus-rila/persistent_peers.txt")"
sed -i -e "s/^peers =.*/peers = \"$peers\"/" $HOME/.nolus/config/config.toml
sed -i 's/max_num_inbound_peers =.*/max_num_inbound_peers = 40/g' $HOME/.nolus/config/config.toml
sed -i 's/max_num_outbound_peers =.*/max_num_outbound_peers = 40/g' $HOME/.nolus/config/config.toml

```
### Pruning
```bash
sed -i -e "s/^pruning *=.*/pruning = \"nothing\"/" $HOME/.nolus/config/app.toml
```
### Disable Indexer (optional) 
```bash
indexer="null"
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.nolus/config/config.toml
```
# Service
```bash
sudo tee /etc/systemd/system/nolusd.service > /dev/null <<EOF
[Unit]
Description=nolusd
After=network-online.target

[Service]
User=$USER
ExecStart=$(which nolusd) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

## Start
```bash
sudo systemctl daemon-reload && sudo systemctl enable nolusd
sudo systemctl restart nolusd && sudo journalctl -u nolusd -f -o cat
```

### Check node logs
```bash
sudo journalctl -fu nolusd -o cat
```
# 4.Start from snapshot
> Snapshots updated every 6 hours
```bash
sudo systemctl stop nolusd
cp $HOME/.nolus/data/priv_validator_state.json $HOME/.nolus/priv_validator_state.json.backup
rm -rf $HOME/.nolus/data $HOME/.nolus/wasm
curl https://lefey.tech/nolus/nolus-snap.tar.gz | tar -zxvf - -C $HOME/.nolus --strip-components 2
curl https://lefey.tech/nolus/nolus-wasm.tar.gz | tar -zxvf - -C $HOME/.nolus --strip-components 2
mv $HOME/.nolus/priv_validator_state.json.backup $HOME/.nolus/data/priv_validator_state.json
sudo systemctl start nolusd && sudo journalctl -fu nolusd -o cat
