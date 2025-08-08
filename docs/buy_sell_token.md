# Buy and Sell token

> Buying and selling depends on wether the token is bonding or already graduated

### Buy token

- ⚠️ Fill up the data at the top (RPC, keypairs, token metadata)
- ⚠️ Install all imported NPM packages

```javascript
// Replace with your Helius RPC URL
const RPC = "https://mainnet.helius-rpc.com/?api-key=???";
// Replace with token address
const TOKEN_ADDRESS = "TOKEN_ADDRESS";
// Replace with PumpSwap pair address. Could be obtained from token's DexScreener page
const PUMPSWAP_PAIR_ADDRESS = "PUMPSWAP_PAIR_ADDRESS";
// Buyer wallet private key. Could be obtained from Phantom wallet. Have some SOL on this wallet
const BUYER_PK = "Private key. Be careful!";
// Amount of SOL you want to buy the token for
const BUY_SOL_AMOUNT = 0.001;

import {
  Keypair,
  Connection,
  sendAndConfirmTransaction,
  Transaction,
  PublicKey,
} from "@solana/web3.js";
import { getBuyTokenAmountFromSolAmount, PumpSdk } from "@pump-fun/pump-sdk";
import {
  PumpAmmSdk,
  PumpAmmInternalSdk,
  buyQuoteInputInternal,
} from "@pump-fun/pump-swap-sdk";
import * as spl from "@solana/spl-token";
import bs58 from "bs58";
import BN from "bn.js";

async function main() {
  const buyer = Keypair.fromSecretKey(bs58.decode(BUYER_PK));
  const mint = new PublicKey(TOKEN_ADDRESS);
  const connection = new Connection(RPC, "confirmed");
  const pumpSdk = new PumpSdk(connection);

  const { bondingCurve, bondingCurveAccountInfo, associatedUserAccountInfo } =
    await pumpSdk.fetchBuyState(new PublicKey(TOKEN_ADDRESS), buyer.publicKey);

  if (bondingCurve.complete) {
    // In case the token has graduated

    const [pumpAmmSdk, pumpAmmInternalSdk] = [
      new PumpAmmSdk(connection),
      new PumpAmmInternalSdk(connection),
    ];
    const swapSolanaState = await pumpAmmSdk.swapSolanaState(
      new PublicKey(PUMPSWAP_PAIR_ADDRESS),
      buyer.publicKey
    );
    const { globalConfig, pool, poolBaseAmount, poolQuoteAmount } =
      swapSolanaState;
    const slippage = 5;

    // Get the associated token address
    const ata = spl.getAssociatedTokenAddressSync(mint, buyer.publicKey, true);
    const ataIx = spl.createAssociatedTokenAccountIdempotentInstruction(
      buyer.publicKey,
      ata,
      buyer.publicKey,
      mint
    );

    const { base } = buyQuoteInputInternal(
      new BN(BUY_SOL_AMOUNT * 1e9),
      slippage,
      poolBaseAmount,
      poolQuoteAmount,
      globalConfig,
      pool.creator
    );

    const buyIxs = await pumpAmmInternalSdk.buyBaseInput(
      swapSolanaState,
      base,
      slippage
    );
    const transaction = new Transaction().add(ataIx, ...buyIxs);
    const signature = await sendAndConfirmTransaction(connection, transaction, [
      buyer,
    ]);

    console.log(`Buy transaction: https://solscan.io/tx/${signature}`);
  } else {
    // In case the token is still bonding

    const global = await pumpSdk.fetchGlobal();
    const solAmount = new BN(BUY_SOL_AMOUNT * 1000000000);
    const amount = getBuyTokenAmountFromSolAmount(
      global,
      bondingCurve,
      solAmount
    );

    // Get the associated token address
    const ata = spl.getAssociatedTokenAddressSync(mint, buyer.publicKey, true);
    const ataIx = spl.createAssociatedTokenAccountIdempotentInstruction(
      buyer.publicKey,
      ata,
      buyer.publicKey,
      mint
    );

    const buyIxs = await pumpSdk.buyInstructions({
      global,
      associatedUserAccountInfo,
      bondingCurve,
      bondingCurveAccountInfo,
      mint,
      user: buyer.publicKey,
      slippage: 15,
      amount,
      solAmount,
    });

    const transaction = new Transaction().add(ataIx, ...buyIxs);

    const signature = await sendAndConfirmTransaction(connection, transaction, [
      buyer,
    ]);

    console.log(`Buy transaction: https://solscan.io/tx/${signature}`);
  }
}

main();
```

### Sell token

- ⚠️ Fill up the data at the top (RPC, keypairs, token metadata)
- ⚠️ Install all imported NPM packages

```javascript
// Replace with your Helius RPC URL
const RPC = "https://mainnet.helius-rpc.com/?api-key=???";
// Replace with token address
const TOKEN_ADDRESS = "TOKEN_ADDRESS";
// Replace with PumpSwap pair address. Could be obtained from token's DexScreener page
const PUMPSWAP_PAIR_ADDRESS = "PUMPSWAP_PAIR_ADDRESS";
// Seller wallet private key. Could be obtained from Phantom wallet. Have some SOL on this wallet
const SELLER_PK = "Private key. Be careful!";
// Amount of tokens you want to sell
const SELL_TOKEN_AMOUNT = 100000;

import {
  Keypair,
  Connection,
  sendAndConfirmTransaction,
  Transaction,
  PublicKey,
} from "@solana/web3.js";
import { getSellSolAmountFromTokenAmount, PumpSdk } from "@pump-fun/pump-sdk";
import {
  PumpAmmSdk,
  PumpAmmInternalSdk,
  sellBaseInputInternal,
} from "@pump-fun/pump-swap-sdk";
import bs58 from "bs58";
import BN from "bn.js";

async function main() {
  const seller = Keypair.fromSecretKey(bs58.decode(SELLER_PK));
  const mint = new PublicKey(TOKEN_ADDRESS);
  const connection = new Connection(RPC, "confirmed");
  const pumpSdk = new PumpSdk(connection);

  const { bondingCurve, bondingCurveAccountInfo } =
    await pumpSdk.fetchSellState(
      new PublicKey(TOKEN_ADDRESS),
      seller.publicKey
    );

  if (bondingCurve.complete) {
    // In case the token has graduated

    const [pumpAmmSdk, pumpAmmInternalSdk] = [
      new PumpAmmSdk(connection),
      new PumpAmmInternalSdk(connection),
    ];
    const swapSolanaState = await pumpAmmSdk.swapSolanaState(
      new PublicKey(PUMPSWAP_PAIR_ADDRESS),
      seller.publicKey
    );
    const { globalConfig, pool, poolBaseAmount, poolQuoteAmount } =
      swapSolanaState;
    const slippage = 5;

    const { uiQuote } = sellBaseInputInternal(
      new BN(SELL_TOKEN_AMOUNT * 1e6),
      slippage,
      poolBaseAmount,
      poolQuoteAmount,
      globalConfig,
      pool.creator
    );

    const sellIxs = await pumpAmmInternalSdk.sellQuoteInput(
      swapSolanaState,
      uiQuote,
      slippage
    );
    const transaction = new Transaction().add(...sellIxs);
    const signature = await sendAndConfirmTransaction(connection, transaction, [
      seller,
    ]);

    console.log(`Sell transaction: https://solscan.io/tx/${signature}`);
  } else {
    // In case the token is still bonding

    const global = await pumpSdk.fetchGlobal();
    const amount = new BN(SELL_TOKEN_AMOUNT * 1000000);
    const solAmount = getSellSolAmountFromTokenAmount(
      global,
      bondingCurve,
      amount
    );

    const sellIxs = await pumpSdk.sellInstructions({
      global,
      bondingCurve,
      bondingCurveAccountInfo,
      mint,
      user: seller.publicKey,
      slippage: 5,
      amount,
      solAmount,
    });

    const transaction = new Transaction().add(...sellIxs);

    const signature = await sendAndConfirmTransaction(connection, transaction, [
      seller,
    ]);

    console.log(`Sell transaction: https://solscan.io/tx/${signature}`);
  }
}

main();
```
