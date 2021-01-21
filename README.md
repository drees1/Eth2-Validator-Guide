# Eth2 Validator Setup Guide

The purpose of this document is to recreate or update my enviroment for Eth2 validation and serve as a resource for other early adopters. 

This follows the general process outlined in [Somer's Guide](https://someresat.medium.com/guide-to-staking-on-ethereum-2-0-ubuntu-prysm-56f681646f74) 

Rather than provisioning a server on AWS or other cloud providers, I opted to buy an Intel NUC on ebay. The cost was less than renting on AWS for one year, I get another machine to play with, and this approach aligns the with decentralized ethos of the network. 

Prerequisites:
- 32 Ether
- Intel NUC with Ubuntu 20 Server installed (Intel i7, 16GB RAM, 500GB SDD)
- Remote computer

This setup will install and configure a full Eth1 node (Go Ethereum), an Eth2 beacon chain (Prysm), and an Eth2 Validator (Prysm), along with monitoring tools for the server.

## Setup Server SSH Connection

Note: These are the only steps that must be completed locally on the server. After SSH is setup, the server can be controlled entirely from a remote computer.

Install openSSH on server:
```sh
$ sudo apt install openssh-server
$ ssh localhost
```

Get the ip address of server <ServerIP> from the output of:
```sh
$ ifconfig
```
        
Login as root and grant admin priveliges to your username
```sh
$ sudo -i
# adduser <UserName>
# usermod -aG sudo <UserName>
```

From remote computer, connect to the server using SSH
```sh
$ ssh <UserName>@<ServerIP>
```

Note: <ServerIP> is dynamically assigned and may change when the server reboots. This creates a problem for establishing ssh connections if the server is not locally accessible. You can modify the server networking config files at `/etc/netplan/*.yaml` to assign static IPs.

# Harden Server SSH Connection

Harden the security of the SSH connection by changing the port number from default (22). The new port number can be between 1024-49151.

On the server, check that the new port number is not already in use:
```sh 
$ sudo ss -tulpn | grep ":<SSHPortNum>"
```

Change the connection port in config files:
```sh
$ sudo nano /etc/ssh/sshd_config
```

Uncomment the `Port 22` line and replace the number with the new port number:
```
Port <SSHPortNum>
```
Save and exit.

Restart SSH service:
```sh
$ sudo systemctl restart ssh
```

The previous ssh command will now fail. Connect to the server with this command from the remote computer:
```sh
$ ssh <UserName>@<ServerIP> -p <SSHPortNum>`
```

# Configure Server Firewall

Install firewall:
```sh
$ sudo apt install ufw
```

Deny inbound traffic, allow outbound traffic
```sh
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
```

Create a rule to deny traffic on the default ssh port (22):
```sh
$ sudo ufw deny 22/tcp
```

Use the following command to create firewall exceptions:
```sh
$ sudo ufw allow <PortNumber>/<Protocol>
```
Create rules for the following ports:

| Port Number  | Protocol | Service	                |
| ------------ | -------- | --------------------------- |
| SSHPortNum   | tcp      | custom SSH port             |
| 30303        |          | geth p2p connections        |
| 13000        | tcp      | prysm p2p connections       |
| 12000        | udp      | prysm p2p connections       |
| 7500         | tcp      | prysm validator node web UI |
| 3500         | tcp      | prysm beacon node web UI    |
| 8080         |          | prysm beacon logs           |
| 8081         |          | prysm validator logs        |
| 9090	       |          | prometheus                  |
| 3000         |          | grafana                     |

Enable the firewall and check status:
```sh
$ sudo ufw enable
$ sudo ufw status numbered
```

Note: for subsequent setup steps requiring repository downloads with `curl` or `apt-get` commands, requests will not pass through the firewall. Disable the firewall with `sudo ufw disable` and re-enable after making the download.

# Configure timekeeping

Check that the timekeeping service in running correctly:
```sh
$ timedatectl
```
The `NTP service` should be `active`. 

If not, run:
```sh
$ sudo timedatectl set-ntp on
```

Remove other timekeeping services from system: 
```sh
$ ntpq -p
$ sudo apt-get remove ntp
```

Set timezone to local time.

# Setup Eth1 Node (Go Ethereum - geth)

Install the Go Ethereum implementation:
```sh
//Disable Firewall
$ sudo apt-get-repository -y ppa:ethereum/ethereum
$ sudo apt update
$ sudo apt install geth
//Enable Firewall
```

Create and configure a user account for the process to run as a background service under, along with a directory to store the Eth1 node data:
```sh
$ useradd --no-create-home --shell /bin/false goeth
$ sudo mkdir -p /var/lib/goethereum
$ sudo chown -R goeth:goeth /var/lib/goethereum
$ sudo nano /etc/system/geth.service
```

Paste the following:
```sh
[Unit]
Description=Go Ethereum Client
After=network.target
Wants=network.target

[Service]
User=goeth
Group=goeth
Type=simple
Restart=always
RestartSec=5
ExecStart=geth --http --datadir /var/lib/goethereum --cache 2048 --maxpeers 30

[Install]
WantedBy=default.target
```
Save and exit.

Configuration notes:
- `--http` exposes and endpoint at `http://localhost:8585` that the prysm beacon node connects to
-  `--cache` sets the size of the internal cache in GB. `2048` uses roughly 4-5GB of memory.
- `--maxpeers` modifies the maximum number of peers that you connect with. More peers uses more internet data, but less peers slows syncing.

Reload the system, start the service, and check status:
```sh
$ sudo systemctl daemon-reload
$ sudo systemctl start geth
$ sudo systemctl enable geth start on reboot
$ sudo systemctl status geth
```

The service should say `active (running)` in green text. Check that the chain is connecting with peers and syncing the chain. Otherwise, troubleshoot networking issues. Syncing may take a couple of days.

Continously moniter log with:
```sh
$ sudo journal ctl -fu geth.service
```

When syncing completes, you should see logs with such:
```
INFO [01-08|11:24:12.169] Imported new chain segment               blocks=1  txs=227  mgas=12.479  elapsed=291.516ms mgasps=42.806  number=11615422 hash="b42040…18c1c5
```

# Setup Eth2 Node (Prysm Beacon Chain - prysmbeacon)

Download the latest prysm beacon chain release, at the time of writing that is v1.0.5:
```sh
\\Disable Firewall
$ curl -LO LO https://github.com/prysmaticlabs/prysm/releases/download/v1.0.5/beacon-chain-v1.0.5-linux-amd64
\\Enable Firewall
```

Make the file execuatable and move to `/usr/local/bin`:
```sh
$ mv beacon-chain-vX.X.X-linux-amd64 beacon-chain
$ chmod +x beacon-chain
$ sudo mv beacon-chain /usr/local/bin
```

Create and configure a user account to the process to run as a background service under, along with a directory to store the Eth2 node data:
```sh
$ sudo useradd --no-create-home --shell /bin/false prysmbeacon
$ sudo mkdir -p /var/lib/prysm/beacon
$ sudo chown -R prysmbeacon:prysmbeacon /var/lib/prysm/beacon
$ sudo chmod 700 /var/lib/prysm/beacon
$ sudo nano /etc/systemd/system/prysmbeacon.service
```

Paste the following into the file:
```
[Unit]
Description=Prysm Eth2 Client Beacon Node
Wants=network-online.target
After=network-online.target

[Service]
User=prysmbeacon
Group=prysmbeacon
Type=simple
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/beacon-chain --datadir=/var/lib/prysm/beacon --http-web3provider=http://127.0.0.1:8545 --accept-terms-of-use --web

[Install]
WantedBy=multi-user.target
```

Configuration notes:
- `http-web3provider` connects prysm Eth2 beacon node with the local geth Eth1 node
- `accept-terms-of-use` is required to start the service

Reload system, start service, enable on reboot, and check status:
```sh
$ sudo systemctl daeomon-reload
$ sudo systemctl start prysmbeacon
$ sudo systemctl enable prysmbeacon
$ sudo systemctl status prysmbeacon
```

The service should say `active (running)` in green text. Check that the chain is connecting with peers and syncing the chain. Otherwise, troubleshoot networking issues. The Eth2 chain is much smaller than the Eth1 chain, so syncing should not take as long.

Continously moniter with:
```
$ sudo journal ctl -fu prysmbeacon.service
```
While syncing, you will see logs with such:
```
msg="Processing deposits from Ethereum 1 chain" deposits=1024 genesisValidators=1019 prefix=powchain
```
When syncing completes, you should see logs with such:
```
msg="Synced new block" block=0x2cbd5999... epoch=8603 finalizedEpoch=8601 finalizedRoot=0x0002023a... prefix=blockchain slot=275325 slotInEpoch=29
```
```
msg="Finished applying state transition" attestations=18 attesterSlashings=0 deposits=0 prefix=blockchain proposerSlashings=0 voluntaryExits=0
```

# Setup Validator (Prysm Validator - prysmvalidator)

Download the latest prysm validator release:
```sh
\\disable firewall
$ curl -LO LO https://github.com/prysmaticlabs/prysm/releases/download/v1.0.5/validator-v1.0.5-linux-amd64
\\enable firewall
```

Make the file execuatable and move to `/usr/local/bin`:
```
$ mv beacon-chain-vX.X.X-linux-amd64 validator
$ chmod +x validator
$ sudo mv validator /usr/local/bin
```

Create validator wallet:
```sh
$ sudo mkdir -p /var/lib/prysm/validator
$ sudo chown -R <UserName>:<UserName> /var/lib/prysm/validator
```

Disconnect from ssh. On remote computer, install the Eth2 deposit command line tool binaries. `$ETH2DEPOSITCLIHOME` is wherever you choose to install it.
```sh
$ cd $ETH2DEPOSITCLIHOME
$ curl -LO https://github.com/ethereum/eth2.0-deposit-cli/releases/download/v1.1.0/eth2deposit-cli-ed5a6d3-linux-amd64.tar.gz
$ tar xvf eth2deposit-cli-ed5a6d3-linux-amd64.tar.gz eth2deposit-cli
```

Generate validator key. TURN OFF INTERNET BEFORE PROCEEDING
```sh
$ cd eth2deposit-cli
$ ./deposit new-mnemonic --num_validators 1 --chain mainnet
```
Write down the mnemonic seed, confirm, and two files will be created in `$ETH2DEPOSITCLIHOME/eth2deposit-cli`: 
- `deposit_data-<TimeStamp>.json` contains the public keys for validators and information for funding the staking deposit contract. You will upload this in the validator launchpad process. 
- `keystore-m<IndexAndTimeStamp>.json` contains the validator private keys. You will upload this to the validator server.

Note: You can generate keys for multiple validators at once, or use `existing-mnemonic` command to generate more validator keys under the same mnemonic later on.

Reconnect to internet, connect to server using STFP to upload the keystore:
```sh
$ stfp -P <SSHPortNum> <UserName>@<ServerIP>
stfp> mkdir /var/lib/eth2deposit-cli/validator_keys
stfp> cd /var/lib/eth2deposit-cli/validator_keys
stfp> put $ETH2DEPOSITCLIHOME/validator_keys/keystore-m<IndexAndTimeStamp>.json
\\Ctrl+D to close stfp connection
```

Reconnect to server via ssh. Import validator keys to service:
```sh
$ cd /usr/local/bin
$ validator accounts import --keys-dir=/var/lib/eth2deposit-cli/validator_keys
```
Accept terms of use and create a password for the wallet.

Create a wallet password file:
```sh
$ sudo nano /var/lib/prysm/validator/password.txt
```
Type the password that you just created, then save and exit.

Create and configure a user account to the process to run as a background service under, along with a directory to store the Eth2 node data:
```sh
$ sudo useradd --no-create-home --shell /bin/false prysmvalidator
$ sudo chown -R prysmvalidator:prysmvalidator /var/lib/prysm/validator
$ sudo chmod 700 /var/lib/prysm/validator
$ sudo chmod -R 700 /var/lib/validator/password.txt
$ sudo nano /etc/systemd/system/prysmbeacon.service
```

Paste the following into the file:
```
[Unit]
Description=Prysm Eth2 Validator Client
Wants=network-online.target
After=network-online.target

[Service]
User=prysmvalidator
Group=prysmvalidator
Type=simple
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/validator --datadir=/var/lib/prysm/validator --wallet-dir=/var/lib/prysm/validator --wallet-password-file=/var/lib/prysm/validator/password.txt --graffiti="<yourgraffiti>" --accept-terms-of-use --web

[Install]
WantedBy=multi-user.target
```
Configuration notes:
- `--graffiti` lets your validator tag the blockchain with a grafitti string. Do use any self-identifying information.
- `--web` exposes endpoints to access the prysm webUI, detailed under "Monitering" section. 

Reload system, start service, enable on reboot, and check status:
```sh
$ sudo systemctl daemon-reload
$ sudo systemctl start prysmvalidator
$ sudo systemctl enable prysmvalidator
$ sudo systemctl status prysmvalidator
```

Continuously access logs with:
```sh
sudo journalctl -fu prysmvalidator.service
```
Log Notes:
- If your beacon chain has not finished syncing, the log will show a `msg` indicating such. 
- You have not yet sent 32 Ether through the deposit contract, so the log will show `waiting to observe deposit`. Make sure that the `pubKey` in the logs matche  s the `pubKey` of your `deposit_data` file. Even after sending the Ether, it may take several hours to observe the deposit.
- After deposit is sent and processed, they will show `Deposit processed, entering activation queue after finalization` until the validator is activated

## Fund Validator Keys

Follow the process detailed in https://launchpad.ethereum.org/.

Upload `deposit_data` file when prompted. Make sure that the public key matches the public key from prysm validator logs.

Proceed with the transaction using a browser wallet (MetaMask).

## Monitering with Prysm web UI

The prysem web UI can be accessed using port forwarding to a remote computer:
```sh
ssh -L 3500:127.0.0.1:3500 -L 7500:127.0.0.1:7500 -L 8080:127.0.0.1:8080 -L 8081:127.0.0.1:8081 <UserName>@<ServerIP> -p <SSHPortNum>
```

Then visit `localhost:7500` in a browser:
```sh
http://localhost:7500
```

While the validator is pending activaion, the UI shows 0 ETH Balance and days in activation queue

## Monitering with Prometheus and Grafana dashboard

Download and unpack prometheus installation files:
```sh
//disable firewall
$ curl -LO url -LO https://github.com/prometheus/prometheus/releases/download/v2.22.0/prometheus-2.22.0.linux-amd64.tar.gz
$ tar -xvf prometheus-2.22.0.linux-amd64.tar.gz
$ mv prometheus-2.22.0.linux-amd64 prometheus-files
//enable firewall
```

Create a user for the prometheus background process:
```sh
$ sudo useradd --no-create-home --shell /bin/false prometheus
$ sudo mkdir /etc/prometheus
$ sudo mkdir /var/lib/prometheus
$ sudo chown prometheus:prometheus /etc/promethus
$ sudo chown prometheus:prometheus /var/lib/promethus
```

Move the binary files into `/usr/local/bin`:
```sh
$ sudo cp prometheus-files/promethus /usr/local/bin/
$ sudo cp prometheus-files/promtool /usr/local/bin/
$ sudo chown prometheus:prometheus /usr/local/bin/
$ sudo chown prometheus:prometheus /usr/local/bin/promtool
```

Move the console files to `/etc/prometheus/`:
```sh
$ sudo cp -r prometheus-files/consoles /etc/promethus
$ sudo cp -r prometheus-files/console_libraries /etc/prometheus
$ sudo chown -R prometheus:prometheus /etc/prometheus/consoles
$ sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
```

Create the configuration file:
```sh
$ sudo nano /etc/prometheus.yml
```

Paste the following into the configuration file:
```sh
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093
# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  - job_name: 'validator'
    static_configs:
      - targets: ['localhost:8081']
  - job_name: 'beacon node'
    static_configs:
      - targets: ['localhost:8080']
  - job_name: 'slasher'
    static_configs:
      - targets: ['localhost:8082']
```

Change ownership of config file:
```sh
$ sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

Create a background service:
```sh
$ sudo nano /etc/systemd/system/prometheus.service
```

Paste the following into the file:
```sh
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

Restart daemon, start prometheus, enable on startup, and check status:
```sh
$ sudo systemctl daemon-reload
$ sudo systemctl start prometheus
$ sudo systemctl enable prometheus
$ sudo systemctl status prometheus
```

The prometheus web UI can be accessed using port forwarding to a remote computer:
```sh
$ ssh -L 9090:127.0.0.1:9090 <UserName>@<ServerIP>
```

Access the web UI at:
```sh
http://localhost:9090/graph
```

Download grafana and install it:
```sh
//disable firewall
$ sudo apt-get install -y adduser libfontconfig1
$ wget https://dl.grafana.com/oss/release/grafana_7.3.7_amd64.deb
$ sudo dpkg -i grafana_7.3.7_amd64.deb
//enable firewall
```

Restart daemon, start grafana, check status, and enable background service on startup:
```sh
$ sudo systemctl daemon-reload
$ sudo systemctl start grafana-server
$ sudo systemctl status grafana-server
$ sudo systemctl enable grafana-server.service
```

The grafana web UI can be accessed from a remote computer using port forwarding:
```sh
$ ssh -L 3000:127.0.0.1:3000 <UserName>@<ServerIP>
```

This visit `localhost:3000` is a browser:
```sh
http://localhost:3000
```
