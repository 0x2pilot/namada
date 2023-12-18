# Namada node with custom ports installation guide
![Namada logo](../image/namada.jpg)
## Overview
Bellow you will find step-by-step instructions on how to install namada node with custom ports. This is useful if you 
have default ports occupied by another tendermint node for example. Guide is updated with the latest testnet
requirements, so fill free to use it to set up node for the current and upcoming testnets.

## Instructions for testnet 15

1. Export variables
    
    Paste you validator name in `ALIAS` variable

    Set `NODE_ID=0` if you don't want to use custom port.
    Set `NODE_ID=1` if you have already 1 tendermint node running on the same machine.
    Set `NODE_ID=2` if you have already 2 tendermint nodes running and etc.

    The rest of parameters will be updated to the required values once the new testnet is announced.
    ```
    export ALIAS="" && \
    export NODE_ID="0" && \
    export CHAIN_ID=public-testnet-15.0dacadb8d663 && \
    export BINARY_URL=https://github.com/anoma/namada/releases/download/v0.28.1/namada-v0.28.1-Linux-x86_64.tar.gz && \
    export BASE_DIR=$HOME/.local/share/namada && \
    export TM_HASH=v0.1.4-abciplus && \
    export RPC_PORT=$((NODE_ID+26))657 && \
    echo custom rpc port used: $RPC_PORT
    ```

2. Update / install binaries

    ```
    cd $HOME && rm -rf namada-v*Linux-x86_64* $HOME/namada-binaries && mkdir $HOME/namada-binaries && wget $BINARY_URL && \
    tar -xf namada-*-Linux-x86_64.tar.gz -C $HOME/namada-binaries --strip-components=1 && cd $HOME/namada-binaries && \
    sudo chmod +x namada* && sudo cp namada* /usr/local/bin/ && cd && namada --version
    ```

3. [ Skip it unless you are installing it the first time ] Install tendermint and go 18

    ```
    git clone https://github.com/heliaxdev/tendermint && cd tendermint && git checkout $TM_HASH && make build && sudo cp build/tendermint /usr/local/bin/ && cd
    ```

4. Download genesis

    ```
    namada client utils join-network --chain-id $CHAIN_ID --genesis-validator $ALIAS
    ```

5. [ Skip if you don't need custom ports ] Setup custom ports

    ```
    sed -i.bak -e "\
    s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:$((NODE_ID+26))658\"%; \
    s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:$((NODE_ID+26))657\"%; \
    s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:$((NODE_ID+26))656\"%; \
    s%^oracle_rpc_endpoint = \"http://127.0.0.1:8545\"%oracle_rpc_endpoint = \"http://127.0.0.1:$((NODE_ID+8))545\"%;" \
    $BASE_DIR/$CHAIN_ID/config.toml
    ```

6. Create / update namada service

```
sudo tee /etc/systemd/system/namadad.service > /dev/null <<EOF
[Unit]
Description=namada
After=network-online.target
[Service]
User=root
WorkingDirectory=$BASE_DIR
Environment=NAMADA_LOG=debug
Environment=NAMADA_CMT_STDOUT=true
ExecStart=/usr/local/bin/namada --base-dir=$BASE_DIR --chain-id $CHAIN_ID node ledger run
StandardOutput=syslog
StandardError=syslog
Restart=always
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

7. Start namada service and see the logs

    ```
    sudo systemctl daemon-reload && sudo systemctl enable namadad && \
    sudo systemctl restart namadad && sudo journalctl -u namadad -fo cat
    ```
   
## Peculiarities of node management with custom ports
When running on custom port any command should have the following flags

`--base-dir "$HOME/.local/share/namada"` for your namada home directory

`--ledger-address localhost:$RPC_PORT` - where `RPC_PORT` is the custom port you are using for your rpc

For example

`namada --base-dir "$HOME/.local/share/namada" client bonds --validator $ALIAS --ledger-address localhost:$RPC_PORT`

## Useful commands

Check node status:
```
curl -s http://127.0.0.1:$RPC_PORT/status | jq
```

Check balance:
```
namada --base-dir "$HOME/.local/share/namada" client balance --owner $ALIAS --token NAM --ledger-address localhost:$RPC_PORT
```

Check bonded stake:

```
namada --base-dir "$HOME/.local/share/namada" client bonded-stake --validator $ALIAS --ledger-address localhost:36657
```