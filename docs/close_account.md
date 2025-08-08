# Closing accounts

> On each `Buy` transaction a `volume accumulator account` gets created. One could return the SOL rent by closing the account.

### Closing volume accumulator account

- ⚠️ Fill up the data at the top (RPC, keypairs, token metadata)
- ⚠️ Install all imported NPM packages

```javascript
// Replace with your Helius RPC URL
const RPC = "https://mainnet.helius-rpc.com/?api-key=???";
// Buyer wallet private key. Could be obtained from Phantom wallet. Have some SOL on this wallet
const BUYER_PK = "Private key. Be careful!";

import {
  Keypair,
  Connection,
  sendAndConfirmTransaction,
  Transaction,
} from "@solana/web3.js";
import { PumpSdk } from "@pump-fun/pump-sdk";
import bs58 from "bs58";

async function main() {
  const buyer = Keypair.fromSecretKey(bs58.decode(BUYER_PK));
  const connection = new Connection(RPC, "confirmed");
  const pumpSdk = new PumpSdk(connection);

  const closeAccountIx = await pumpSdk.closeUserVolumeAccumulator(
    buyer.publicKey
  );
  const transaction = new Transaction().add(closeAccountIx);
  const signature = await sendAndConfirmTransaction(connection, transaction, [
    buyer,
  ]);

  console.log(`Close account transaction: https://solscan.io/tx/${signature}`);
}

main();
```
