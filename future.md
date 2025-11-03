# Production Integration Guide

## Quick Start: Refactoring for dApp Integration

### Step 1: Extract Core Logic into a Library

The current code can be refactored into a reusable library. Here's how:

#### Current Structure Issues:
- Functions print to console instead of returning values
- Uses global `settings` object (tight coupling)
- Synchronous blocking operations
- No wallet abstraction

#### Recommended Refactoring:

### Python SDK Approach (For Backend Services)

Create `src/sdk/` directory with clean interfaces:

```python
# src/sdk/base.py

from abc import ABC, abstractmethod
from typing import Protocol, Optional, Callable
from web3 import Web3
from xrpl.clients import JsonRpcClient

class FlareWallet(Protocol):
    """Protocol for Flare wallet operations"""
    def get_address(self) -> str: ...
    def sign_transaction(self, tx: dict) -> str: ...
    def send_transaction(self, tx: dict) -> str: ...

class XRPLWallet(Protocol):
    """Protocol for XRPL wallet operations"""
    def get_address(self) -> str: ...
    def sign_payment(self, payment: dict) -> dict: ...
    def send_payment(self, payment: dict) -> str: ...

class SmartAccountsClient:
    """Clean SDK interface without CLI dependencies"""
    
    def __init__(
        self,
        flare_rpc_url: str,
        xrpl_rpc_url: str,
        flare_wallet: FlareWallet,
        xrpl_wallet: XRPLWallet
    ):
        self.w3 = Web3(Web3.HTTPProvider(flare_rpc_url))
        self.xrpl = JsonRpcClient(xrpl_rpc_url)
        self.flare_wallet = flare_wallet
        self.xrpl_wallet = xrpl_wallet
        # Load registry
        from src.registry import Registry
        self.registry = Registry.default()
        self.encoder = encoder  # From src.encoder
    
    async def deposit(
        self,
        amount: int,
        on_progress: Optional[Callable[[str], None]] = None
    ) -> dict:
        """
        Deposit FXRP into vault.
        
        Returns:
            {
                "xrpl_tx_hash": str,
                "flare_tx_hash": str,
                "personal_account": str
            }
        """
        # Encode instruction
        memo_data = self.encoder.deposit(amount).hex()
        
        # Send XRPL transaction
        if on_progress:
            on_progress("Sending XRPL transaction...")
        xrpl_tx_hash = await self._send_bridge_request(memo_data)
        
        # Wait for bridging
        if on_progress:
            on_progress("Waiting for bridge...")
        bridged_event = await self._wait_for_bridged(xrpl_tx_hash)
        
        return {
            "xrpl_tx_hash": xrpl_tx_hash,
            "flare_tx_hash": bridged_event["transactionHash"].hex(),
            "personal_account": bridged_event["args"]["personalAccount"]
        }
    
    async def _send_bridge_request(self, memo: str) -> str:
        """Send bridge request transaction"""
        # Get operator address from contract
        operator_address = (
            self.w3.eth.contract(
                address=self.registry.master_account_controller.address,
                abi=self.registry.master_account_controller.abi,
            )
            .functions.xrplProviderWallet()
            .call()
        )
        
        # Create and sign payment
        payment = {
            "amount": "1",
            "destination": operator_address,
            "memos": [{"memo_data": memo}]
        }
        
        return await self.xrpl_wallet.send_payment(payment)
    
    async def _wait_for_bridged(
        self,
        xrpl_tx_hash: str,
        timeout: int = 360
    ) -> dict:
        """Wait for XRPL transaction to be bridged to Flare"""
        # Get XRPL transaction timestamp
        xrpl_tx = await self.xrpl.request(Tx(transaction=xrpl_tx_hash))
        timestamp = ripple_time_to_posix(xrpl_tx.result["tx_json"]["date"])
        
        # Estimate block number
        start_block = await self._find_block_near_timestamp(timestamp - 90)
        
        # Wait for InstructionExecuted event
        event = self.registry.master_account_controller.events["InstructionExecuted"]
        
        for block in range(start_block, start_block + timeout):
            await asyncio.sleep(1)
            current_block = self.w3.eth.block_number
            
            if current_block < block:
                continue
            
            logs = self.w3.eth.get_logs({
                "address": event.contract.address,
                "topics": [event.signature],
                "fromBlock": block,
                "toBlock": current_block
            })
            
            for log in logs:
                decoded = get_event_data(self.w3.codec, event.abi, log)
                if decoded["args"]["transactionId"].hex() == xrpl_tx_hash.lower():
                    return decoded
            
            if current_block > start_block + timeout:
                raise TimeoutError("Bridge timeout")
    
    # ... other methods
```

### Step 2: Create Wallet Adapters

```python
# src/sdk/wallets/metamask.py (Example for backend)

class MetaMaskFlareWallet:
    """Adapter for MetaMask wallet (used via API)"""
    
    def __init__(self, api_client):
        self.api = api_client
    
    def get_address(self) -> str:
        return self.api.request_account()
    
    def sign_transaction(self, tx: dict) -> str:
        return self.api.sign_transaction(tx)
```

```python
# src/sdk/wallets/env_wallet.py

class EnvironmentWallet:
    """Adapter for CLI-style environment variable wallets"""
    
    def __init__(self, private_key: str):
        self.pk = private_key
    
    def get_address(self) -> str:
        account = w3.eth.account.from_key(self.pk)
        return account.address
    
    def sign_transaction(self, tx: dict) -> str:
        signed = w3.eth.account.sign_transaction(tx, self.pk)
        return signed.raw_transaction.hex()
```

### Step 3: TypeScript SDK for Frontend

For dApps, you'll want a TypeScript SDK. Here's the structure:

```typescript
// packages/js-sdk/src/index.ts

export { SmartAccountsSDK } from './sdk';
export { MetaMaskAdapter } from './wallet-adapters/metamask';
export { XUMMAdapter } from './wallet-adapters/xumm';
export { WalletConnectAdapter } from './wallet-adapters/walletconnect';
export * from './types';

// packages/js-sdk/src/encoder.ts

export class Encoder {
  deposit(amount: bigint): string {
    const instructionId = 1;
    const value = amount.toString(16).padStart(62, '0');
    return `${instructionId.toString(16).padStart(2, '0')}${value}`;
  }
  
  withdraw(amount: bigint): string {
    const instructionId = 2;
    const value = amount.toString(16).padStart(62, '0');
    return `${instructionId.toString(16).padStart(2, '0')}${value}`;
  }
  
  // ... other encoders
}
```

### Step 4: React Integration Example

```typescript
// examples/react-example/src/components/Deposit.tsx

import { useSmartAccount } from '../hooks/useSmartAccount';
import { useState } from 'react';

export function DepositForm() {
  const { deposit } = useSmartAccount();
  const [amount, setAmount] = useState('');
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState<any>(null);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    
    try {
      const result = await deposit(BigInt(amount), {
        onProgress: (status) => console.log(status)
      });
      setResult(result);
    } catch (error) {
      console.error('Deposit failed:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="number"
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
        placeholder="Amount (drops)"
      />
      <button type="submit" disabled={loading}>
        {loading ? 'Processing...' : 'Deposit'}
      </button>
      {result && (
        <div>
          <p>XRPL TX: {result.xrplTxHash}</p>
          <p>Flare TX: {result.flareTxHash}</p>
        </div>
      )}
    </form>
  );
}
```

## Recommended Package Structure

### Monorepo Setup

```
smart-accounts/
├── packages/
│   ├── js-sdk/              # TypeScript SDK (npm package)
│   │   ├── src/
│   │   │   ├── sdk.ts
│   │   │   ├── encoder.ts
│   │   │   ├── flare-client.ts
│   │   │   ├── xrpl-client.ts
│   │   │   └── wallet-adapters/
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   └── python-sdk/          # Python SDK (PyPI package)
│       ├── src/
│       │   └── smart_accounts_sdk/
│       │       ├── __init__.py
│       │       ├── sdk.py
│       │       ├── encoder.py
│       │       └── wallets/
│       ├── pyproject.toml
│       └── setup.py
│
├── examples/
│   ├── react-example/
│   ├── vue-example/
│   └── python-backend-example/
│
└── cli/                     # Keep CLI as separate tool
    └── smart_accounts.py
```

## Migration Checklist

- [ ] Extract encoder logic (already pure, easy to port)
- [ ] Refactor clients to use dependency injection
- [ ] Remove print statements, return structured data
- [ ] Add async/await support
- [ ] Create wallet adapter interfaces
- [ ] Add comprehensive error handling
- [ ] Write TypeScript SDK
- [ ] Add React hooks for common patterns
- [ ] Create example applications
- [ ] Publish to npm and PyPI
- [ ] Write integration documentation

## Quick Implementation Priority

1. **Immediate**: Extract Python SDK from CLI code
2. **High Priority**: Create TypeScript SDK for frontend
3. **Medium**: Add wallet adapter support (MetaMask, Xumm)
4. **Nice to Have**: React hooks, Vue composables

## Usage in Production

### Backend Service (Python)

```python
from smart_accounts_sdk import SmartAccountsClient, EnvironmentWallet

# Initialize
client = SmartAccountsClient(
    flare_rpc_url="https://coston2-api.flare.network/ext/C/rpc",
    xrpl_rpc_url="wss://s.altnet.rippletest.net:51233",
    flare_wallet=EnvironmentWallet(os.environ["FLR_PRIVATE_KEY"]),
    xrpl_wallet=EnvironmentWallet(os.environ["XRPL_SECRET"])
)

# Use it
result = await client.deposit(1000000)  # 1 FXRP
print(f"Deposited: {result['flare_tx_hash']}")
```

### Frontend dApp (TypeScript/React)

```typescript
import { SmartAccountsSDK } from '@flare-foundation/smart-accounts-sdk';
import { MetaMaskAdapter } from '@flare-foundation/smart-accounts-sdk/wallets';
import { XUMMAdapter } from '@flare-foundation/smart-accounts-sdk/wallets';

const sdk = new SmartAccountsSDK(
  {
    flareRpcUrl: 'https://coston2-api.flare.network/ext/C/rpc',
    network: 'coston2'
  },
  new MetaMaskAdapter(window.ethereum),
  new XUMMAdapter()
);

const result = await sdk.deposit(BigInt(1000000));
console.log('Deposited!', result.flareTxHash);
```

