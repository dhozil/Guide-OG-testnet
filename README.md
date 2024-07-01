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
git clone -b v0.2.3 https://github.com/0glabs/0g-chain.git
cd 0g-chain
make install
```

### Init Moniker
```
0gchaind config chain-id zgtendermint_16600-2
0gchaind init YOUR-MONIKER --chain-id zgtendermint_16600-2
```

### Set up configuration
Prepare the genesis file And set some mandatory configuration options
```
sudo apt install -y unzip wget
rm ~/.0gchain/config/genesis.json
wget -P ~/.0gchain/config https://github.com/0glabs/0g-chain/releases/download/v0.2.3/genesis.json
```

### Set minimum gas price & peers
```
sed -i "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0ua0gi\"/" $HOME/.0gchain/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.0gchain/config/config.toml
```
```
# Update peers in the config
PEERS="5a9aac3b111f8ef78da298d747f6f79daf2b5954@31.220.75.10:12656,8609834e666efda4caf41f3f2fa293d1df39a495@38.242.240.234:12656,cadf176a1a7cc769cb2e614728c5455bbc4c9be1@158.220.125.5:12656,c311c0e00ba7a8d998a57cbdb77c769279e5d79b@161.97.152.80:56656,df4cc52fa0fcdd5db541a28e4b5a9c6ce1076ade@37.60.246.110:13456,6e044d233c4abb2cc970c8fc2e968273c38a874e@167.86.116.237:12656,0dade47457780be569d7e5be9eedcf6731ed2d18@5.189.133.249:12656,87050b88e0dff2df18caff484e01c32d9f6e6a49@185.209.223.108:12656,d7921529d985b18096ea5cc5d023806af91fd51e@157.90.128.250:58656,55982724a7a30944215ad45924071f1efc1eef4a@116.202.174.53:26856,5ba403bf2183ffbc2aea2508af82041ad69cb883@195.201.242.245:12656,9d06dc7b225e7c32146896d5fec3d91ffe1c5395@94.130.137.49:26656,6a07fd41680eacfd29b63c7ce07a0f20af18bfa8@193.233.75.244:26656,3b3ddcd4de429456177b29e5ca0febe4f4c21989@75.119.139.198:26656,4577c3d8be80ca946da72f138f0c7d1d311f9be6@31.220.73.247:26656" && \
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.0gchain/config/config.toml
```

### Set pruning
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "10"|' \
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
  --chain-id=zgtendermint_16600-2 \
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
