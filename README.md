# Guide OG Labs Testnet Node

## Testnet Setup Instructions

### Install dependencies

Update system and install build tools
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget build-essential jq make lz4 gcc unzip -y
```

### Install GO
```
cd $HOME && \
ver="1.21.5" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile && \
go version
```

### Install binary
```
cd $HOME && mkdir -p go/bin/
git clone -b v0.1.0 https://github.com/0glabs/0g-chain.git
./0g-chain/networks/testnet/install.sh
source .profile
```

### Init Moniker
```
0gchaind config chain-id zgtendermint_16600-1
0gchaind init YOUR-MONIKER --chain-id zgtendermint_16600-1
```

### Set up configuration
Prepare the genesis file And set some mandatory configuration options
```
rm ~/.0gchain/config/genesis.json
curl -Ls https://github.com/0glabs/0g-chain/releases/download/v0.1.0/genesis.json > $HOME/.0gchain/config/genesis.json
curl -Ls https://raw.githubusercontent.com/Core-Node-Team/Testnet-TR/main/0G-Newton/addrbook.json > $HOME/.0gchain/config/addrbook.json
```
```
rm ~/.0gchain/config/genesis.json
wget -P ~/.0gchain/config https://github.com/0glabs/0g-chain/releases/download/v0.1.0/genesis.json
```

### Set minimum gas price & peers
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0agnet"|g' $HOME/.0gchain/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.0gchain/config/config.toml
```
```
# Update peers in the config
PEERS="5ee69001a49e9fad7f60cad99f0a91d823810757@82.67.49.126:37000,d50aa74398af77c6c7bb5483bb1760989b780b8d@109.123.236.150:26656,213a99e681bcd8160ea1d209b8298c86e0355e67@94.156.117.229:26656,4b37705126b4cf304265e9d31c982b39bfb76c9a@94.156.117.19:26656,a41a2ef37d4bd5886f25010815cea53c7bd105cb@109.199.123.253:26656,6220bc5afb26f02627d13819b282aadad14f07c1@38.242.201.36:26656,a6076b5d78b9b37fd3488af51f2b9dcc6978f9e8@185.11.251.182:47656,ffa48d99f37eaf5473a02487ef26fd7691f770aa@149.102.141.113:26656,74fd185131f9903c694dec71a17b5def4f5d6fc1@80.71.227.186:26656,149f92d947d99aaadf2f1a01f510d528915b1952@45.94.209.90:26656,0be430094f421c136200fd2353f3082a0de0997f@62.171.144.252:16656,3bc22d3f241e2a292b583f06bc875f272ec2873e@207.244.244.26:26656,23ca14ea5c8f4dc37815f2108bd3b5157945a638@161.97.73.109:16656,da69988481b49b9e6865620a1296366bf36aeb11@213.199.39.178:26656,eb5053d264804ec65922065f888393ed38f6264c@5.78.79.198:16656" && \
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.0gchain/config/config.toml
```

### Set pruning
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.0gchain/config/app.toml
```

### Create service file
```
sudo tee /etc/systemd/system/0gchaind.service > /dev/null <<EOF
[Unit]
Description=0gchaind
After=network-online.target

[Service]
User=$USER
ExecStart=$(which 0gchaind) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

### Enable and start service
```
sudo systemctl daemon-reload
sudo systemctl enable 0gchaind  
sudo systemctl restart 0gchaind  && sudo journalctl -fu 0gchaind  -o cat
```

## Key management

### Add new wallet
```
0gchaind keys add wallet
```

### Restore wallet
```
0gchaind keys add wallet --recover
```

### Check wallet balance 
```
0gchaind q bank balances $(0gchaind keys show wallet -a)
```

### Create validator
```
0gchaind tx staking create-validator \
  --amount=900000ua0gi \
  --pubkey=$(0gchaind tendermint show-validator) \
  --moniker="YOUR-MONIKER" \
  --chain-id=zgtendermint_16600-1 \
  --commission-rate=0.05 \
  --commission-max-rate=0.1 \
  --commission-max-change-rate=0.1 \
  --min-self-delegation=1 \
  --from=wallet \
  --identity="" \
  --details="" \
  --website="" \
  --gas=500000 --gas-prices=99999ua0gi
  -y
```

## Additional commands

### Validator info
```
0gchaind q staking validator $(0gchaind keys show wallet --bech val -a)
```

### Node info
```
0gchaind status 2>&1 | jq
```

### Delegate to your validator
```
0gchaind tx staking delegate $(0gchaind keys show WALLET_NAME --bech val -a) 1000000ua0gi --from wallet -y
```

### Unjail Validator 
```
0gchaind tx slashing unjail --from WALLET_NAME --gas=500000 --gas-prices=99999neuron -y
```

### Stop Node
```
sudo systemctl stop 0gchaind
```

### Restart Node
```
sudo systemctl restart 0gchaind
```

### Check Log
```
sudo journalctl -u 0gchaind -f -o cat
```

### Disable Indexing
```
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.0gchain/config/config.toml
```

### Delete Node
```
sudo systemctl stop 0gchaind.service
sudo systemctl disable 0gchaind.service
sudo rm /etc/systemd/system/0gchaind.service
rm -rf $HOME/.0gchain $HOME/0g-chain
```
