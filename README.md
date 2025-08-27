# Private Sepolia and Beacon RPC Setup Guide
This repo provides a simple, step-by-step tutorial to set up and run your own private RPC nodes for the Sepolia Ethereum testnet (execution layer) and Beacon chain (consensus layer). This allows you to have local, private access to RPC endpoints without relying on public providers like Infura or Alchemy.

## Why do this?
- Reliability: Run your own node for faster, customized access.
- Privacy: Keep your queries and transactions off public nodes.
- Learning: Understand Ethereum's architecture.

## Requirements:
### Hardware:
- Ubuntu 22.04 LTS or similar Linux OS (recommended; Windows/macOS possible but more complex).
- RAM: 16 GB
- CPU: 4-6 cores
- Disk: 1TB+ SSD (NVMe preferred for faster syncing).
- Stable internet with unmetered bandwidth

### Access: 
 - Root or sudo privileges on your VPS or local machine

#### Warning:
Running nodes syncs the blockchain, which can take hours/days and use significant bandwidth/storage. Start with Sepolia (testnet) to avoid real ETH costs. Keep RPC private – do not expose ports publicly without firewalls.

# Step-by-Step Guide

### Step 1: Initial Setup
#### 1. Update the System:
- Ensure your system is up-to-date:

      sudo apt update && sudo apt upgrade -y

#### 2. Install Dependencies:
- Install required packages for building and running the node:

      sudo apt install -y curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip

#### 3. Install Docker
- Docker simplifies running Geth and Prysm. Install it with:

      sudo apt update -y && sudo apt upgrade -y
      sudo apt-get install -y ca-certificates curl gnupg
      sudo install -m 0755 -d /etc/apt/keyrings
      curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
      sudo chmod a+r /etc/apt/keyrings/docker.gpg
      echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
      sudo apt update -y
      sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

- Verify Docker installation:

      sudo docker run hello-world

    This should output a confirmation message. If it fails, check your Docker installation.

#### 4. Enable Docker:
- Ensure Docker starts on boot:

      sudo systemctl enable docker
      sudo systemctl restart docker

### Step 2: Create Directories and JWT Secret
#### 1. Create Directories:
- Set up directories for Geth (execution client) and Prysm (consensus client):

      mkdir -p /root/ethereum/execution
      mkdir -p /root/ethereum/consensus

#### 2. Generate JWT Secret:
- The JWT secret secures communication between Geth and Prysm. Generate it with:

      openssl rand -hex 32 > /root/ethereum/jwt.hex

- Verify it was created:

      cat /root/ethereum/jwt.hex

    You should see a 32-byte hexadecimal string.

### Step 3: Configure Docker Compose
#### 1. Create `docker-compose.yml`:
- Navigate to the Ethereum directory:

      cd /root/ethereum
      nano docker-compose.yml

- Paste the following configuration into `docker-compose.yml`:

      version: '3.8'
      services:
        geth:
          image: ethereum/client-go:stable
          container_name: geth
          network_mode: host
          restart: unless-stopped
          ports:
            - 30303:30303
            - 30303:30303/udp
            - 8545:8545
            - 8546:8546
            - 8551:8551
          volumes:
            - /root/ethereum/execution:/data
            - /root/ethereum/jwt.hex:/data/jwt.hex
          command:
            - --sepolia
            - --http
            - --http.api=eth,net,web3
            - --http.addr=0.0.0.0
            - --authrpc.addr=0.0.0.0
            - --authrpc.vhosts=*
            - --authrpc.jwtsecret=/data/jwt.hex
            - --authrpc.port=8551
            - --syncmode=snap
            - --datadir=/data
          logging:
            driver: "json-file"
            options:
              max-size: "10m"
              max-file: "3"
        prysm:
          image: gcr.io/prysmaticlabs/prysm/beacon-chain
          container_name: prysm
          network_mode: host
          restart: unless-stopped
          volumes:
            - /root/ethereum/consensus:/data
            - /root/ethereum/jwt.hex:/data/jwt.hex
          depends_on:
            - geth
          ports:
            - 4000:4000
            - 3500:3500
          command:
            - --sepolia
            - --accept-terms-of-use
            - --datadir=/data
            - --disable-monitoring
            - --rpc-host=0.0.0.0
            - --execution-endpoint=http://127.0.0.1:8551
            - --jwt-secret=/data/jwt.hex
            - --rpc-port=4000
            - --grpc-gateway-corsdomain=*
            - --grpc-gateway-host=0.0.0.0
            - --grpc-gateway-port=3500
            - --min-sync-peers=3
            - --checkpoint-sync-url=https://checkpoint-sync.sepolia.ethpandaops.io
            - --genesis-beacon-api-url=https://checkpoint-sync.sepolia.ethpandaops.io
          logging:
            driver: "json-file"
            options:
              max-size: "10m"
              max-file: "3"

- Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

#### 2. Explanation of Configuration:
- **Geth**: Runs the execution client for Sepolia, exposing:
   - HTTP RPC endpoint on port 8545 (`eth`, `net`, `web3` APIs).
   - WebSocket RPC on port 8546.
   - Auth RPC on port 8551 (for Prysm communication).
   - P2P port 30303 for network connectivity.

- **Prysm**: Runs the Beacon chain (consensus client), exposing:
   - HTTP API on port 3500 (for Beacon chain queries).
   - RPC on port 4000.
   - Uses a checkpoint sync URL to speed up Beacon chain syncing.

 - **JWT**: Shared secret ensures secure communication between Geth and Prysm.
 - **Volumes**: Persist data in `/root/ethereum/execution` and `/root/ethereum/consensus`.

### Step 4: Check Port Availability
- Ensure the required ports (30303, 8545, 8546, 8551, 4000, 3500) are free:

      sudo apt install -y net-tools
      netstat -tuln | grep -E '30303|8545|8546|8551|4000|3500'

  - If any ports are in use, stop the conflicting services or change the ports in docker-compose.yml.

### Step 5: Set Up Firewall
#### 1. Install and Configure UFW:
- Install Uncomplicated Firewall (UFW):

      sudo apt install -y ufw

- Allow necessary ports:

      sudo ufw allow 22/tcp  # SSH
      sudo ufw allow 30303/tcp  # Geth P2P
      sudo ufw allow 30303/udp  # Geth P2P
      sudo ufw allow 8545/tcp   # Geth HTTP RPC
      sudo ufw allow 3500/tcp   # Prysm HTTP API

  - **Security Note**: For production, restrict port 8545 to specific IPs (e.g., your local machine or Aztec node IP) to prevent public access:

      sudo ufw allow from YOUR_TRUSTED_IP to any port 8545 proto tcp

  Replace `YOUR_TRUSTED_IP` with your IP address.
  - Enable the firewall:

      sudo ufw enable
      sudo ufw status

### Step 6: Start Geth and Prysm
#### 1. Run the Nodes
  - Start both Geth and Prysm using Docker Compose:

      cd /root/ethereum
      docker compose up -d

  - The `-d` flag runs containers in the background.

#### 2. Check Logs:
  - Monitor logs to ensure the nodes are running:

      docker compose logs -fn 100

  - Press `Ctrl+C` to stop viewing logs.
  - **Note**: Prysm syncs quickly (a few hours) using the checkpoint sync URL. Geth may take up to 48 hours to fully sync the Sepolia blockchain.

### Step 7: Verify Node Sync Status
#### 1. Check Geth Sync Status:
  - Run the following command to check if Geth is synced:

        curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' http://localhost:8545

  - **Output**:
    - If syncing: You’ll see details like `currentBlock` and `highestBlock`.
    - If synced: `{"jsonrpc":"2.0","id":1,"result":false}`

#### 2. Check Prysm Sync Status:
  - Check if the Beacon chain is synced:

        curl http://localhost:3500/eth/v1/node/syncing

  - **Output**:
    - If synced: `{"data":{"head_slot":"12345","sync_distance":"0","is_syncing":false}}`


### Step 8: Get RPC Endpoints
Once both nodes are synced, you can use the following endpoints for your node:
  - **Geth (Execution Layer) RPC**:
    - Inside VPS: `http://localhost:8545`
    - Outside VPS: `http://<your-vps-ip>:8545` (e.g., `http://101.1.101.1:8545`)
    - **Aztec Sequencer Execution RPC**: Use the above endpoint in your Aztec node configuration (e.g., via CLI or config file).
  - **Security Note**: If exposing endpoints externally (outside the VPS), ensure firewall rules (Step 5) restrict access to trusted IPs to prevent unauthorized use.


## Additional Tips
 - Sync Time: Geth may take 24-48 hours to sync. Prysm syncs faster with the checkpoint URL. Be patient during initial setup.
 - Security: Never expose port 8545 publicly without IP whitelisting, as it allows transaction submissions and can be abused.
 - Backup: Regularly back up `/root/ethereum` to preserve node data.

## Why This Setup?
 - Running your own Sepolia and Beacon chain nodes avoids rate limits from providers like Alchemy or Infura, which is critical for Aztec Network’s sequencer or development needs.
 - The Docker-based setup simplifies dependency management and ensures reproducibility.
 - The provided endpoints (`http://<your-vps-ip>:8545` for execution, `http://<your-vps-ip>:3500` for Beacon) are compatible with Aztec’s requirements for private, high-performance RPC access.

