---
layout: post
title: "Creating a Custom Token on Solana: From Concept to Mainnet"
date: 2025-12-06
description: "How I built KYORO token with metadata, TypeScript tooling, and a cat mascot - for under $0.02"
tags: [solana, blockchain, cryptocurrency, typescript, web3]
---

*How I built KYORO token with metadata, TypeScript tooling, and a cat mascot - for under $0.02*

Creating your own cryptocurrency sounds intimidating, but on Solana, it's surprisingly straightforward and affordable. This is the story of how I created KYORO, a utility token that will power my Open-Lotto gaming protocol, complete with custom metadata and a personal touch.

## Why Create KYORO?

I needed a token for my lottery protocol that wouldn't fall under gambling regulations. By creating a gaming token with no monetary value (distributed free via airdrops), users can enjoy the lottery mechanics without real money changing hands. Plus, I wanted to document the entire token creation process for other developers entering the Solana ecosystem.

## The Technical Journey

### Step 1: Token Creation

The basic token creation on Solana is refreshingly simple:
```bash
# Ensure you're on mainnet
solana config set --url https://api.mainnet-beta.solana.com

# Create the token mint
spl-token create-token
```

**Output:**
```
Creating token AHgVJscCvsGR7jWX39VRFKvyZU1jjaNcARHawYsinM6d
Address:  AHgVJscCvsGR7jWX39VRFKvyZU1jjaNcARHawYsinM6d
Decimals:  9
```

**Cost:** ~0.00001 SOL (~$0.002)

This creates a token mint account on Solana with zero supply. The address `AHgVJscCvsGR7jWX39VRFKvyZU1jjaNcARHawYsinM6d` is now KYORO's permanent identifier.

### Step 2: Creating Your Token Account

Before you can hold tokens, you need an Associated Token Account (ATA):
```bash
spl-token create-account AHgVJscCvsGR7jWX39VRFKvyZU1jjaNcARHawYsinM6d
```

This creates a PDA (Program Derived Address) specifically for holding KYORO tokens in your wallet.

### Step 3: Minting Tokens

Now for the magic moment - creating tokens out of thin air:
```bash
spl-token mint AHgVJscCvsGR7jWX39VRFKvyZU1jjaNcARHawYsinM6d 1000000
```

**Result:** 1 million KYORO tokens now exist, all in my wallet.

## The Metadata Challenge

At this point, KYORO worked perfectly as a token, but wallets would only display the raw mint address. To show "KYORO" with a custom image, I needed to add metadata using Metaplex.

This is where things got interesting.

### Setting Up TypeScript Tooling

Rather than wrestling with command-line tools, I built a TypeScript project to handle metadata:
```bash
mkdir kyoro-token-metadata
cd kyoro-token-metadata
npm init -y
npm install @solana/web3.js @metaplex-foundation/mpl-token-metadata \
  @metaplex-foundation/umi @metaplex-foundation/umi-bundle-defaults \
  typescript ts-node
```

The key was configuring `package.json` for ESM modules:
```json
{
  "type": "module"
}
```

### The Personal Touch: A Cat Mascot

I decided to use a photo of my cat as KYORO's token icon. Using Mac's built-in Preview app, I:
- Cropped the image to square (512x512 pixels)
- Saved as PNG
- Uploaded to GitHub for permanent hosting

### Creating the Metadata JSON

Metaplex requires a JSON file with token information:
```json
{
  "name": "KYORO",
  "symbol": "KYORO",
  "description": "A utility token on the Solana blockchain",
  "image": "https://raw.githubusercontent.com/arussel/arussel.github.io/main/assets/images/2c5b0950-ebe7-410a-a50c-dd0bb9db9f73.png",
  "external_url": "https://arussel.github.io"
}
```

I hosted this on GitHub at: `https://raw.githubusercontent.com/arussel/kyoro-token-metadata/main/kyoro-metadata.json`

### The Metadata Creation Script

Here's the TypeScript script that creates the metadata account:
```typescript
import { createUmi } from '@metaplex-foundation/umi-bundle-defaults';
import { keypairIdentity, publicKey } from '@metaplex-foundation/umi';
import { createV1, TokenStandard } from '@metaplex-foundation/mpl-token-metadata';

const MINT_ADDRESS = 'AHgVJscCvsGR7jWX39VRFKvyZU1jjaNcARHawYsinM6d';

async function addMetadata() {
  const umi = createUmi('https://api.mainnet-beta.solana.com');
  
  // Load wallet keypair
  const keypairPath = path.join(process.env.HOME!, '.config/solana/id.json');
  const secretKey = JSON.parse(fs.readFileSync(keypairPath, 'utf8'));
  const keypair = umi.eddsa.createKeypairFromSecretKey(new Uint8Array(secretKey));
  
  umi.use(keypairIdentity(keypair));
  
  const createMetadataIx = createV1(umi, {
    mint: publicKey(MINT_ADDRESS),
    authority: keypair,
    name: 'KYORO',
    symbol: 'KYORO',
    uri: '', // Initially empty
    sellerFeeBasisPoints: 0,
    tokenStandard: TokenStandard.Fungible,
  });
  
  const result = await createMetadataIx.sendAndConfirm(umi);
  console.log('Metadata created!');
}
```

## The Gotcha: Verified Creators

The initial metadata creation worked, but KYORO still showed no image in wallets. The issue? I had set the URI to empty string.

When I tried to update the metadata with the proper JSON URI, I hit a wall:
```
Error: Verified creators cannot be removed.
```

The Metaplex program had automatically set me as a verified creator, and its security model prevents removing verified creators - even by the creator themselves.

### The Solution

The fix required explicitly preserving the creator structure when updating:
```typescript
const updateMetadataIx = updateV1(umi, {
  mint: publicKey(MINT_ADDRESS),
  authority: keypair,
  data: {
    name: 'KYORO',
    symbol: 'KYORO',
    uri: 'https://raw.githubusercontent.com/arussel/kyoro-token-metadata/main/kyoro-metadata.json',
    sellerFeeBasisPoints: 0,
    creators: [
      {
        address: publicKey('CmidxXKPyWhqpDEch8VQoTuDpHSzfmpFDXGRWQuUXLRX'),
        verified: true,
        share: 100,
      }
    ],
    collection: null,
    uses: null,
  }
});
```

Success! KYORO now displays with name, symbol, and cat image across all Solana wallets and explorers.

## Real-World Considerations

### Spam Protection

Phantom wallet initially flagged KYORO as spam - a sensible default for unknown tokens. This required manually marking it as "not spam" in wallet settings.

### Total Costs

The entire process cost approximately $0.015:

| Operation | Cost (SOL) | Cost (USD) |
|-----------|------------|------------|
| Token creation | ~0.00001 | ~$0.002 |
| Token account | ~0.00001 | ~$0.002 |
| Minting tokens | ~0.00001 | ~$0.002 |
| Initial metadata | ~0.00001 | ~$0.002 |
| Metadata update | ~0.00001 | ~$0.002 |
| **Total** | **~0.00005 SOL** | **~$0.012** |

Compare this to deploying an ERC-20 token on Ethereum mainnet, which can cost $50-200 in gas fees.

### Legal Considerations

By keeping the description generic ("A utility token on the Solana blockchain") and distributing for free, KYORO avoids securities regulations that might apply to tokens with investment implications.

## Key Learnings

1. **Solana token creation is remarkably cheap and fast** - the entire process took a few hours and cost pennies.

2. **Metadata is optional but essential** - tokens work fine without it, but users expect names and images.

3. **TypeScript tooling works well** once you handle ESM configuration properly.

4. **GitHub makes excellent permanent hosting** for token metadata and images.

5. **Verified creators add security but complicate updates** - plan your metadata carefully from the start.

6. **Real-world deployment has gotchas** that tutorials don't mention - spam filters, wallet caching, metadata propagation delays.

## Development Challenges

### TypeScript Module Configuration

Getting the right module configuration was tricky. The key was:
- Setting `"type": "module"` in `package.json`
- Configuring `tsconfig.json` for ESNext modules
- Using `ts-node` with ESM support

### Metaplex SDK Evolution

The Metaplex ecosystem has multiple CLI tools and SDKs:
- Sugar CLI (for NFT collections)
- Old `@metaplex-foundation/js` (deprecated)
- New Umi SDK (current standard)

Documentation often references outdated packages, making it challenging to find the right approach.

### Metadata Propagation

Even after successful transactions, metadata doesn't appear immediately across all platforms:
- Solscan updated within minutes
- Phantom required app restart
- Some explorers took longer to show images

## What's Next

KYORO will be integrated into my [Open-Lotto protocol](https://github.com/arussel/open-lotto) as a gaming token. Users will receive KYORO through airdrops and use it to participate in lottery games without real money gambling.

Future plans include:
- Airdrop distribution mechanism
- Integration with Open-Lotto smart contracts
- Frontend for token distribution

## Resources

- **KYORO Token:** [View on Solscan](https://solscan.io/token/AHgVJscCvsGR7jWX39VRFKvyZU1jjaNcARHawYsinM6d)
- **Source Code:** [kyoro-token-metadata repository](https://github.com/arussel/kyoro-token-metadata)
- **Metadata JSON:** [kyoro-metadata.json](https://github.com/arussel/kyoro-token-metadata/blob/main/kyoro-metadata.json)
- **My Portfolio:** [arussel.github.io](https://arussel.github.io)

## Conclusion

Creating KYORO demonstrated how accessible blockchain development has become. With less than $0.02 and a few hours of work, I created a fully-featured cryptocurrency with custom branding. The barrier to entry for building on Solana is remarkably low - the only limit is your imagination.

The experience taught me that while the basic mechanics are simple, the devil is in the details. Real-world deployment involves tooling challenges, security considerations, and user experience factors that tutorials often gloss over. But that's what makes it interesting - and what makes documenting the complete journey valuable for other developers.

---

*KYORO token (AHgVJscCvsGR7jWX39VRFKvyZU1jjaNcARHawYsinM6d) was created for educational and utility purposes. This is not financial advice.*