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

## Step 1: Initial Setup
1. Update your system:
``` sudo apt update && sudo apt upgrade -y

2. Install Dependencies:
``` sudo apt install -y curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip





