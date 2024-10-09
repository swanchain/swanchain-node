# SwanChain Node Installation Guide

This guide will walk you through the process of setting up a SwanChain node.

## Prerequisites

Ensure you have `wget` installed on your system.

## 1. Download Required Files

Run the following commands to download the necessary files:

```bash
wget -O genesis.json https://chaindata.swanchain.org/genesis.json
wget -O rollup.json https://chaindata.swanchain.org/rollup.json
wget -O op-node https://chaindata.swanchain.org/op-node
wget -O geth https://chaindata.swanchain.org/geth
```

## 2. Install Binaries

Move the downloaded binaries to the appropriate location and make them executable:

```bash
mv op-node /usr/bin/op-node && chmod +x /usr/bin/op-node
mv geth /usr/bin/geth && chmod +x /usr/bin/geth
```

## 3. Set Up Directories

Create the necessary directories and copy files:

```bash
mkdir -p /root/op-geth /root/op-node
cp genesis.json /root/op-geth
cp rollup.json /root/op-node
```

## 4. Initialize Geth

Initialize the Geth datadir:

```bash
cd /root/op-geth
geth init --datadir=datadir genesis.json
openssl rand -hex 32 > jwt.txt
cp /root/op-geth/jwt.txt /root/op-node/jwt.txt
```

## 7. Create Geth SystemD Service

Create a new systemd service file for Geth:

```bash
sudo vim /etc/systemd/system/geth-swanchain.service
```

Add the following content to the file:

```ini
[Unit]
Description=Geth SwanChain Node
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/op-geth
ExecStart=/usr/bin/geth \
    --override.ecotone=1718640216 \
    --override.cancun=1718640216 \
    --override.canyon=1718640216 \
    --bootnodes 'enode://ed760daf5639d8f635d77e608eff8e9eb54d8d92be8c097654bc7727cdc0e7092017c3cd8e5af6677d11e26e9067ccbd4ddcd6285bf17485c4871e15925666b4@38.80.81.43:30303,enode://e1af88d816ab647479639dc70e2850d2c03bd88093a75c154e5b9a7f216e6d82b4591010ef0f343ed646cf7c4528014ddbb5575fe37ba2555f3fce9389913198@38.80.81.42:30303,enode://c69b87db28ff086699a7ad036a3e577ae86d5a3a9abaa45ca3f7a7980015c31ab828043b13f217c0fc7159eebe126c8a6d21f6c120b7a34e7d51a5f86fcd6891@38.80.81.44:30303' \
    --datadir ./datadir \
    --http \
    --http.corsdomain="*" \
    --http.vhosts="*" \
    --http.addr=0.0.0.0 \
    --http.api=web3,debug,eth,txpool,net,engine \
    --ws \
    --ws.addr=0.0.0.0 \
    --ws.port=8546 \
    --ws.origins="*" \
    --ws.api=debug,eth,txpool,net,engine \
    --syncmode=full \
    --gcmode=archive \
    --networkid=254 \
    --authrpc.vhosts="*" \
    --authrpc.addr=0.0.0.0 \
    --authrpc.port=8551 \
    --authrpc.jwtsecret=./jwt.txt \
    --rollup.disabletxpoolgossip=true \
    --rollup.sequencerhttp=https://sequencer-mainnet.swanchain.io
Restart=always
RestartSec=5
StandardOutput=append:/root/op-geth/geth.log
StandardError=append:/root/op-geth/geth.log

[Install]
WantedBy=multi-user.target
```

Save and close the file.

## 8. Create OP-Node SystemD Service

First, set up the environment variables:

```bash
# Add these lines to /etc/environment
# L1 rpc needs you to make changes, you can get it from quicknode or other L1 providers
echo 'L1_RPC="***"' | sudo tee -a /etc/environment
echo 'L1_Beacon_RPC="***"' | sudo tee -a /etc/environment

# Reload the environment variables
source /etc/environment
```

Now, create a new systemd service file for OP-Node:

```bash
sudo vim /etc/systemd/system/op-node-swanchain.service
```

Add the following content to the file:

```ini
[Unit]
Description=OP-Node SwanChain
After=network.target geth-swanchain.service

[Service]
Type=simple
User=root
WorkingDirectory=/root/op-node
EnvironmentFile=/etc/environment
ExecStart=/usr/bin/op-node \
    --override.ecotone=1718640216 \
    --override.delta=1718640216 \
    --override.canyon=1718640216 \
    --l2=ws://localhost:8551 \
    --l2.jwt-secret=./jwt.txt \
    --verifier.l1-confs=4 \
    --rollup.config=./rollup.json \
    --p2p.static=/ip4/38.80.81.43/tcp/9003/p2p/16Uiu2HAmCEL64AYE62ZWXLc3t3So3fLPAF6T6GLJ23PW26ddZK99,/ip4/38.80.81.42/tcp/9003/p2p/16Uiu2HAkzid7aQdxv8xnNou2rYgFjvNUqXQfs9FaH32ZJFVvzWxA,/ip4/38.80.81.44/tcp/9003/p2p/16Uiu2HAmRbSjKUDeYgKWx7AVPGdGXrRNKCvxbMMtz3WkpJXPNhz3 \
    --l1.trustrpc=true \
    --p2p.listen.ip=0.0.0.0 \
    --p2p.listen.tcp=9003 \
    --p2p.listen.udp=9003 \
    --rpc.addr=0.0.0.0 \
    --rpc.port=8547 \
    --l1=${L1_RPC} \
    --l1.rpckind=quicknode \
    --syncmode=execution-layer \
    --l1.beacon=${L1_Beacon_RPC}
Restart=always
RestartSec=5
StandardOutput=append:/root/op-node/node.log
StandardError=append:/root/op-node/node.log

[Install]
WantedBy=multi-user.target
```

Save and close the file.

## 9. Enable and Start Services

After creating the service files, follow these steps:

1. Reload the systemd manager configuration:
   ```bash
   sudo systemctl daemon-reload
   ```

2. Enable the services to start on boot:
   ```bash
   sudo systemctl enable geth-swanchain.service
   sudo systemctl enable op-node-swanchain.service
   ```

3. Start the services:
   ```bash
   sudo systemctl start geth-swanchain.service
   sudo systemctl start op-node-swanchain.service
   ```

4. Check the status of the services:
   ```bash
   sudo systemctl status geth-swanchain.service
   sudo systemctl status op-node-swanchain.service
   ```

You can now manage the services using the following commands:
- Start: `sudo systemctl start [service-name]`
- Stop: `sudo systemctl stop [service-name]`
- Restart: `sudo systemctl restart [service-name]`
- View logs: `sudo journalctl -u [service-name]`

Replace `[service-name]` with either `geth-swanchain.service` or `op-node-swanchain.service` as needed.

This setup will run both Geth and OP-Node as system services, making them easier to manage and ensuring they start automatically on system boot.

## 9. Monitoring and Maintenance

To monitor the node's performance and sync status:

1. Check Geth logs:
   ```bash
   sudo tail -f /root/op-geth/geth.log
   ```

2. Check OP-Node logs:
   ```bash
   sudo tail -f /root/op-node/node.log
   ```

3. To check the sync status, you can use the Geth console:
   ```bash
   geth attach /root/op-geth/datadir/geth.ipc
   ```
   Then in the Geth console:
   ```javascript
   eth.syncing
   ```

Remember to keep your system updated and monitor for any announcements from the SwanChain team regarding updates or maintenance requirements.

## 10. Troubleshooting

### Q1: How do I change the RPC service port?

A1: If you need to modify the RPC service port, you should change the port number in the Geth service configuration. The default port is 8545.

To change the RPC port:

1. Edit the Geth service file:
   ```bash
   sudo vim /etc/systemd/system/geth-swanchain.service
   ```

2. In the `ExecStart` section, find the `--http.port` flag (if it exists) or add it if it's not present. Set it to your desired port number. For example:
   ```
   --http.port=8545
   ```

3. Save the file and exit the editor.

4. Reload the systemd manager configuration:
   ```bash
   sudo systemctl daemon-reload
   ```

5. Restart the Geth service:
   ```bash
   sudo systemctl restart geth-swanchain.service
   ```

Remember that if you change the RPC port, you may need to update any applications or services that connect to your node to use the new port number.

### Q2: What should I do if the node is not syncing?

A2: If your node is not syncing, try the following steps:

1. Check the Geth and OP-Node logs for any error messages:
   ```bash
   sudo journalctl -u geth-swanchain -n 100
   sudo journalctl -u op-node-swanchain -n 100
   ```

2. Ensure that your system clock is accurate:
   ```bash
   sudo timedatectl status
   ```
   If it's not synchronized, you may need to install and configure NTP.

3. Verify that your firewall is not blocking the required ports (30303 for P2P, 8545 for RPC, etc.).

4. Check your internet connection and ensure you have enough bandwidth.

5. If problems persist, try restarting both services:
   ```bash
   sudo systemctl restart geth-swanchain.service
   sudo systemctl restart op-node-swanchain.service
   ```

If you're still experiencing issues after trying these steps, consider reaching out to the SwanChain community or support channels for further assistance.

### Q3: How can I view the full logs for Geth and OP-Node?

A3: Since we've configured the services to write logs to files, you can view them using the following commands:

For Geth logs:
```bash
sudo less /root/op-geth/geth.log
```

For OP-Node logs:
```bash
sudo less /root/op-node/node.log
```

Use the arrow keys to navigate through the log file. Press 'q' to exit.

If you want to follow the logs in real-time, you can use the `tail` command with the `-f` option:

For Geth:
```bash
sudo tail -f /root/op-geth/geth.log
```

For OP-Node:
```bash
sudo tail -f /root/op-node/node.log
```

Press Ctrl+C to stop following the logs.

If you prefer to use `journalctl` for log viewing, you can modify the service files to use standard output instead of writing to files. To do this:

1. Edit the service files:
   ```bash
   sudo vim /etc/systemd/system/geth-swanchain.service
   sudo vim /etc/systemd/system/op-node-swanchain.service
   ```

2. Remove or comment out the `StandardOutput` and `StandardError` lines in both files.

3. Save the files and exit the editor.

4. Reload the systemd manager and restart the services:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart geth-swanchain.service
   sudo systemctl restart op-node-swanchain.service
   ```

After making these changes, you can use `journalctl` to view the logs:
```bash
sudo journalctl -u geth-swanchain -f
sudo journalctl -u op-node-swanchain -f
```