# Ethereum Archive Node: Development & Production SRE Challenge

This repository demonstrates both **local development** of an Ethereum Archive Node and a **production-grade operational plan** for deploying it on Azure VMs with enterprise tooling.

---

## Local Development Environment

This is your sandbox for understanding how Ethereum nodes work before going to production.

### Local Tech Stack
*   **Execution Client**: [Geth](https://geth.ethereum.org/) (Archive Mode, Syncmode: Full)
*   **Consensus Client**: [Prysm](https://docs.prylabs.network/) (Beacon + Validator)
*   **Orchestration**: Docker Compose
*   **Monitoring**: Prometheus + Grafana
*   **Network**: Local Private PoS Devnet (Interop Mode)

### Quick Start

**Prerequisites:**
*   Docker & Docker Compose
*   `curl` (for testing)

**Launch the Network:**
```bash
# 1. Start all services (Genesis auto-generated)
docker-compose up -d --build

# 2. Check logs to verify block production
docker-compose logs -f beacon
```
*Wait for "Synced new block" logs in the beacon chain and "Chain head was updated" in Geth.*

**Test the Node:**

*Linux/Mac (bash):*
```bash
# Query account balance via JSON-RPC (returns hex value in wei)
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_getBalance","params":["0x123463a4b065722e99115d6c222f267d9cabb524", "0x0"],"id":1}' http://localhost:8545

# Convert hex balance to readable ETH
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_getBalance","params":["0x123463a4b065722e99115d6c222f267d9cabb524", "0x0"],"id":1}' http://localhost:8545 | python3 -c "import sys, json; result = json.load(sys.stdin)['result']; print(f'Balance: {int(result, 16) / 10**18} ETH')"
```

*Windows PowerShell:*
```powershell
# Query account balance via JSON-RPC (returns hex value in wei)
Invoke-RestMethod -Uri http://localhost:8545 -Method POST -ContentType "application/json" -Body '{"jsonrpc":"2.0","method":"eth_getBalance","params":["0x123463a4b065722e99115d6c222f267d9cabb524", "0x0"],"id":1}'

# Convert hex balance to readable ETH
$response = Invoke-RestMethod -Uri http://localhost:8545 -Method POST -ContentType "application/json" -Body '{"jsonrpc":"2.0","method":"eth_getBalance","params":["0x123463a4b065722e99115d6c222f267d9cabb524", "0x0"],"id":1}'; $hex = $response.result -replace '^0x', ''; $balance = [System.Numerics.BigInteger]::Parse($hex, [System.Globalization.NumberStyles]::AllowHexSpecifier) / [System.Numerics.BigInteger]::Parse('1000000000000000000'); Write-Host "Balance: $balance ETH"
```

**Monitoring Dashboard:**
1. Open Grafana at [http://localhost:3000](http://localhost:3000) (admin/admin)
2. Dashboards are **auto-provisioned**:
   - **Geth Dashboard** (Execution Layer metrics)
   - **Prysm Validator Dashboard** (Consensus Layer metrics)

# Inspecting Archive Queries

## Query historical state at specific block

*Linux/Mac (bash):*
```bash
curl -X POST http://localhost:8545 -H "Content-Type: application/json" -d '{"jsonrpc":"2.0","method":"eth_getBalance","params":["0x123463a4b065722e99115d6c222f267d9cabb524", "latest"],"id":1}'
```

*Windows PowerShell:*
```powershell
Invoke-RestMethod -Uri http://localhost:8545 -Method POST -ContentType "application/json" -Body '{"jsonrpc":"2.0","method":"eth_getBalance","params":["0x123463a4b065722e99115d6c222f267d9cabb524", "latest"],"id":1}'
```

*Note: Replace the address as needed. Use `"latest"` for the latest block, `"0x0"` for genesis block, or a specific block number like `"0x1000"` for block 4096.*

## Verify archive mode (should return full history)

*Linux/Mac (bash):*
```bash
curl -X POST http://localhost:8545 -H "Content-Type: application/json" -d '{"jsonrpc":"2.0","method":"eth_getBlockByNumber","params":["0x1", true],"id":1}'
```

*Windows PowerShell:*
```powershell
Invoke-RestMethod -Uri http://localhost:8545 -Method POST -ContentType "application/json" -Body '{"jsonrpc":"2.0","method":"eth_getBlockByNumber","params":["0x1", true],"id":1}'
```


**Note:** Blocks are produced every 2 seconds (configured in `config/config.yaml` with `SECONDS_PER_SLOT: 2`). For faster testing, you can reduce this value. For production-like behavior, set it to `12`. After changing the config, restart the services with `docker-compose restart beacon validator`.

---
# References & Inspiration

*   [Google SRE Book](https://sre.google/sre-book/) - SLOs, error budgets, on-call practices
*   [Ethereum Archive Node Guide](https://www.cherryservers.com/blog/ethereum-archive-node) - Hardware benchmarks
*   [Geth Documentation](https://geth.ethereum.org/docs) - Official Geth docs
*   [Lighthouse Book](https://lighthouse-book.sigmaprime.io/) - Lighthouse consensus client
*   [Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/architecture/) - Best practices for Azure
