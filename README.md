**Manual Installation**
Official Documentation
Recommended Hardware:
```
Intel CPU which supports SGX via SPS, 32GB RAM, 500 GB SSD
```
**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```


**install go, if needed**
```
cd $HOME
VER="1.21.3"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

**set vars**
```
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export SWISS_CHAIN_ID="swisstronik_1291-1"" >> $HOME/.bash_profile
echo "export SWISS_PORT="44"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
rm swisstronikd.zip
wget https://github.com/SigmaGmbH/swisstronik-chain/releases/download/testnet-v1.0.6/swisstronikd.zip
rm -rf bin
unzip swisstronikd.zip
sudo cp ~/bin/libsgx_wrapper_v1.0.6.x86_64.so /usr/lib
cp bin/v1.0.6_enclave.signed.so $HOME/.swisstronik-enclave
sudo mv $HOME/bin/swisstronikd $HOME/go/bin
```

**config and init app**
```
swisstronikd config node tcp://localhost:${SWISS_PORT}657
swisstronikd config keyring-backend os
swisstronikd config chain-id swisstronik_1291-1
swisstronikd init "test" --chain-id swisstronik_1291-1
```

**download genesis and addrbook**
```
wget -O $HOME/.swisstronik/config/genesis.json https://server-2.itrocket.net/testnet/swisstronik/genesis.json
wget -O $HOME/.swisstronik/config/addrbook.json  https://server-2.itrocket.net/testnet/swisstronik/addrbook.json
```

**set seeds and peers**
```
SEEDS="ec00cbf5c72ad261e28fd8de61d09cc2936456ed@swisstronik-testnet-seed.itrocket.net:44656"
PEERS="f05c4343d2df801ba05a5ec7bd9954d8728fdb36@swisstronik-testnet-peer.itrocket.net:26656,6a3be625de39e3db02e6cda719d8c564dfde8086@148.113.16.99:26656,575d7c50fc6ec6a8b233659551499a6ece864bd0@57.128.193.157:26656,3cb2105b9ae008f0711ca5d2e285485b6c3ad1ec@148.113.8.228:26656,1f35bf4128576d94c99297a2e33b06b7ee0ae3d2@146.59.111.161:26656,b368e2232e4cdec602c96b77505401f94a643847@148.113.1.150:17156,d5e1bb92c3c264124f7accbd3f7e6a472401f256@148.113.17.9:26656,4e5574f195f4dc6d0252a37867b951226561647d@57.129.28.2:26656,a9a1aedec8b3a8da921afaa7a7ca6e828207f963@57.128.193.118:23756,1d6ed28a0cd141d402c4e9f69137c6e0541ef1b8@148.113.16.43:26656,faf98ecdbaba68f0c8483618ca9f2842b374031c@146.59.110.154:26656,f1ce897ca531618465ff03caf8de4bc406fd01dc@212.8.240.56:26656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.swisstronik/config/config.toml
```


**set custom ports in app.toml**
```
sed -i.bak -e "s%:1317%:${SWISS_PORT}317%g;
s%:8080%:${SWISS_PORT}080%g;
s%:9090%:${SWISS_PORT}090%g;
s%:9091%:${SWISS_PORT}091%g;
s%:8545%:${SWISS_PORT}545%g;
s%:8546%:${SWISS_PORT}546%g;
s%:6065%:${SWISS_PORT}065%g" $HOME/.swisstronik/config/app.toml
```

**set custom ports in config.toml file**
```
sed -i.bak -e "s%:26658%:${SWISS_PORT}658%g;
s%:26657%:${SWISS_PORT}657%g;
s%:6060%:${SWISS_PORT}060%g;
s%:26656%:${SWISS_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${SWISS_PORT}656\"%;
s%:26660%:${SWISS_PORT}660%g" $HOME/.swisstronik/config/config.toml
```

**config pruning**
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.swisstronik/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.swisstronik/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.swisstronik/config/app.toml
```

**set minimum gas price, enable prometheus and disable indexing**
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "7uswtr"|g' $HOME/.swisstronik/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.swisstronik/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.swisstronik/config/config.toml
```

**create service file**
```
sudo tee /etc/systemd/system/swisstronikd.service > /dev/null <<EOF
[Unit]
Description=Swisstronik node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.swisstronik
ExecStart=$(which swisstronikd) start --home $HOME/.swisstronik
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

**reset and download snapshot**
```
swisstronikd tendermint unsafe-reset-all --home $HOME/.swisstronik
if curl -s --head curl https://server-2.itrocket.net/testnet/swisstronik/swisstronik_2024-10-17_7993421_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-2.itrocket.net/testnet/swisstronik/swisstronik_2024-10-17_7993421_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.swisstronik
    else
  echo "no snapshot founded"
fi
```

**enable and start service**
```
sudo systemctl daemon-reload
sudo systemctl enable swisstronikd
sudo systemctl restart swisstronikd && sudo journalctl -u swisstronikd -f
Automatic Installation
pruning: custom: 100/0/19 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/swisstronik/autoinstall/)
```

Create wallet

**to create a new wallet, use the following command. don’t forget to save the mnemonic**
```
swisstronikd keys add $WALLET
```

**to restore exexuting wallet, use the following command**
```
swisstronikd keys add $WALLET --recover
```

**save wallet and validator address**
```
WALLET_ADDRESS=$(swisstronikd keys show $WALLET -a)
VALOPER_ADDRESS=$(swisstronikd keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**check sync status, once your node is fully synced, the output from above will print "false"**
```
swisstronikd status 2>&1 | jq 
```

**before creating a validator, you need to fund your wallet and check balance**
```
swisstronikd query bank balances $WALLET_ADDRESS 
Node Sync Status Checker
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.swisstronik/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://swisstronik-testnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

  if ! [[ "$local_height" =~ ^[0-9]+$ ]] || ! [[ "$network_height" =~ ^[0-9]+$ ]]; then
    echo -e "\033[1;31mError: Invalid block height data. Retrying...\033[0m"
    sleep 5
    continue
  fi

  blocks_left=$((network_height - local_height))
  if [ "$blocks_left" -lt 0 ]; then
    blocks_left=0
  fi

  echo -e "\033[1;33mNode Height:\033[1;34m $local_height\033[0m \033[1;33m| Network Height:\033[1;36m $network_height\033[0m \033[1;33m| Blocks Left:\033[1;31m $blocks_left\033[0m"

  sleep 5
```

**Create validator**

```
swisstronikd tx staking create-validator \
--amount 1000000uswtr \
--from $WALLET \
--commission-rate 0.1 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--pubkey $(swisstronikd tendermint show-validator) \
--moniker "test" \
--identity "" \
--website "" \
--details "I love blockchain ❤️" \
--chain-id swisstronik_1291-1 \
--gas-adjustment 1.4 --gas auto --gas-prices 7aswtr \
-y
```

Monitoring
If you want to have set up a monitoring and alert system use our cosmos nodes monitoring guide with tenderduty

Security
To protect you keys please don`t share your privkey, mnemonic and follow basic security rules

Set up ssh keys for authentication
You can use this guide to configure ssh authentication and disable password authentication on your server

**Firewall security**

Set the default to allow outgoing connections, deny all incoming, allow ssh and node p2p port
```
sudo ufw default allow outgoing 
sudo ufw default deny incoming 
sudo ufw allow ssh/tcp 
sudo ufw allow ${SWISS_PORT}656/tcp
sudo ufw enable
```

**Delete node**
```
sudo systemctl stop swisstronikd
sudo systemctl disable swisstronikd
sudo rm -rf /etc/systemd/system/swisstronikd.service
sudo rm $(which swisstronikd)
sudo rm -rf $HOME/.swisstronik
sed -i "/SWISS_/d" $HOME/.bash_profile
```
Commit 2 line
Commit 3 line
Commit 4 line
Commit 5 line
Commit 6 line
Commit 7 line
Commit 8 line
Commit 9 line
Commit 10 line
Commit 1 line
Commit 2 line
Commit 3 line
Commit 4 line
Commit 5 line
Commit 6 line
Commit 7 line
Commit 8 line
Commit 9 line
Commit 10 line
Commit 11 line
Commit 12 line
Commit 13 line
Commit 14 line
Commit 15 line
