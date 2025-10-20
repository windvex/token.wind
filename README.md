# 🪙 Token.Wind Smart Contract

Official open-source **Vexanium** token contract used across the **Wind Stack** ecosystem.  
This repository includes ready-to-deploy sources, compiled artifacts (`.wasm`, `.abi`), and developer instructions.  
It can be built with the **VexChain compiler toolchain**, which is designed specifically for the Vexanium blockchain.

---

## 📦 Overview

This contract implements a standard fungible token system inspired by `eosio.token`, optimized for the **Wind Stack DApp** ecosystem.

**Main features:**
- Create and issue fungible tokens  
- Standard transfer and retire actions  
- Open / close balance entries  
- Compatible with `@wharfkit/antelope` and Wind Stack DApp tools  
- Includes pre-built **ABI** and **WASM** binaries for direct integration

---

## ⚙️ Build Tool — VexChain

[VexChain](https://github.com/vexanium/VexChain) is the official compiler and toolchain for **Vexanium smart contracts**.  
It provides a modern, optimized `eosio-cpp` replacement compatible with VEX Leap environments.

### 🔧 Installation (Linux / Ubuntu)

```bash
# Clone VexChain & Manual Build
git clone https://github.com/vexanium/VexChain.git
cd VexChain

# Install dependencies and build
./scripts/eosio_build.sh
./scripts/eosio_install.sh
```

### 🏗️ Compile the contract

Once VexChain is installed:

```bash
eosio-cpp -abigen -I include -contract token -o token.wasm src/token.wind.cpp
```

> ⚠️ **Warnings about Ricardian clauses** are informational only.  
> They indicate missing documentation clauses and do not affect functionality.

---

## 📘 Deploying to Vexanium

1. **Set contract:**
   ```bash
   cleos set contract token.wind /path/to/token.wind/ -p token.wind@active
   ```

2. **Create a token:**
   ```bash
   cleos push action token.wind create '["issueracc", "1000000000.0000 WIND"]' -p token.wind@active
   ```

3. **Issue tokens:**
   ```bash
   cleos push action token.wind issue '["useracc", "100.0000 WIND", "Initial issue"]' -p issueracc@active
   ```

4. **Transfer tokens:**
   ```bash
   cleos push action token.wind transfer '["useracc", "otheracc", "10.0000 WIND", "test"]' -p useracc@active
   ```

---

## 📂 Directory Structure

```
token.wind/
├── include/
│   └── token.wind.hpp        # Header file (contract definition)
├── src/
│   └── token.wind.cpp        # Main contract implementation
├── token.abi                 # ABI (Application Binary Interface)
├── token.wasm                # Compiled WASM bytecode
└── wallet.txt                # (optional) local test wallet / notes
```

---

## 🧩 Integration in Wind Stack

The contract’s ABI and WASM are directly consumable by:
- **Wind Wallet**  
- **Wind DApp** (token, staking, and DEX modules)  
- **Wind Explorer** (ABI decoding, transaction tracing)

Developers can use Antelope formatters to integrate:

```js
import { Asset } from '@wharfkit/antelope'
```

---

## 📜 License

Released under the **MIT License**.  
You are free to use, modify, and distribute this contract with proper attribution.

---

## 💬 Credits

- **Author:** Gilang Ramadan (Founder of Wind)  
- **Ecosystem:** [Wind Stack](https://windcrypto.com)  
- **Blockchain:** [Vexanium Mainnet](https://explorer.vexanium.com)  
- **Compiler Tool:** [VexChain](https://github.com/vexanium/VexChain)

---