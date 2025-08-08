# Create token

## Token address

Token address could be a random one or begin with specific characters you choose (f.e **COOL**92V8G299VuhEV4CpfRrfmXm5qrSwF9w3mn2upump)

> To generate a vanity/specific address, install [Solana CLI](https://solana.com/docs/intro/installation#install-the-solana-cli) and follow [instructions](https://solana.com/developers/cookbook/wallets/generate-vanity-address)

### Token creation code:

- ⚠️ Fill up the data at the top (RPC, keypairs, token metadata)
- ⚠️ Install all imported NPM packages
- ⚠️ Create "img" folder and put token JPEG there

> Run the code below with `npx tsx example.ts`

```javascript
// example.ts

// Replace with your Helius RPC URL
const RPC = "https://mainnet.helius-rpc.com/?api-key=???";
// Replace with generated vanity keypair from ~/.config/solana/id.json or leave as is for random address
const GENERATED_TOKEN_KEYPAIR = [123, 456, 789];
// Deployer's wallet private key. Could be obtained from Phantom wallet. Have some SOL on this wallet
const DEPLOYER_SK = "Private key. Be careful!"
// Amount of tokens you want to buy at the launch
const DEV_BUY_AMOUNT = 1000
const TOKEN_METADATA = {
    name: "TEST TOKEN",
    symbol: "",
    description: "",
    twitter: "",
    telegram: "",
    website: "",
}

import { Keypair, Connection, sendAndConfirmTransaction, Transaction } from "@solana/web3.js";
import { bondingCurvePda, getBuySolAmountFromTokenAmount, PumpSdk } from "@pump-fun/pump-sdk";
import * as spl from "@solana/spl-token";
import axios from "axios";
import bs58 from "bs58";
import BN from "bn.js";
import fs from "fs";

async function main() {
    let mint: Keypair;

    if (GENERATED_TOKEN_KEYPAIR.length == 3) {
        mint = Keypair.generate();
    } else {
        mint = Keypair.fromSecretKey(new Uint8Array(GENERATED_TOKEN_KEYPAIR));
    }

    const deployer = Keypair.fromSecretKey(
        bs58.decode(
            DEPLOYER_SK,
        ),
    );

    const files = await fs.promises.readdir("./img");

    if (files.length == 0) {
        console.log("No image found in the img folder");
        return;
    }

    const data = fs.readFileSync(`./img/${files[0]}`);

    let formData = new FormData();
    if (data) {
        formData.append("file", new Blob([new Uint8Array(data)], { type: "image/jpeg" }));
    } else {
        console.log("No image found");
        return;
    }

    (Object.keys(TOKEN_METADATA) as Array<keyof typeof TOKEN_METADATA>).map(field => formData.append(field, TOKEN_METADATA[field]))

    formData.append("showName", "true");

    let metadata_uri;

    try {
        const response = await axios.post("https://pump.fun/api/ipfs", formData, {
            headers: {
                "Content-Type": "multipart/form-data",
            },
        });

        metadata_uri = response.data.metadataUri;

        console.log("Metadata URI: ", metadata_uri);
    } catch (error) {
        console.error("Error uploading metadata:", error);
    }

    const connection = new Connection(RPC, "confirmed");

    let sdk = new PumpSdk(connection)

    let createIx = await sdk.createInstruction(
        { mint: mint.publicKey, name: TOKEN_METADATA.name, symbol: TOKEN_METADATA.symbol, uri: metadata_uri, creator: deployer.publicKey, user: deployer.publicKey }
    );

    // Get the associated token address
    const ata = spl.getAssociatedTokenAddressSync(mint.publicKey, deployer.publicKey, true);
    const ataIx = spl.createAssociatedTokenAccountIdempotentInstruction(deployer.publicKey, ata, deployer.publicKey, mint.publicKey);

    const global = await sdk.fetchGlobal();

    // Virtual BC for new tokens
    const virtualBondingCurve = {
        virtualTokenReserves: global.initialVirtualTokenReserves,
        virtualSolReserves: global.initialVirtualSolReserves,
        realTokenReserves: global.initialRealTokenReserves,
        realSolReserves: new BN(0),
        tokenTotalSupply: global.tokenTotalSupply,
        complete: false,
        creator: deployer.publicKey,
    };

    // Calculate SOL amount based on tokenAmount
    const amount = new BN(DEV_BUY_AMOUNT * 1000000);
    const solAmount = getBuySolAmountFromTokenAmount(global, virtualBondingCurve, amount)

    let buyIx = await sdk.buyInstructions({
        global: await sdk.fetchGlobal(),
        bondingCurveAccountInfo: {
            ...(await connection.getAccountInfo(
                bondingCurvePda(mint.publicKey),
            ))!, data: new Buffer([])
        },
        bondingCurve: virtualBondingCurve,
        associatedUserAccountInfo: null,
        mint: mint.publicKey,
        user: deployer.publicKey,
        amount,
        solAmount,
        slippage: 1
    })

    const transaction = new Transaction().add(createIx, ataIx, ...buyIx);

    const signature = await sendAndConfirmTransaction(
        connection,
        transaction,
        [deployer, mint]
    );

    console.log("Transaction signature:", signature);
}

main()
```
