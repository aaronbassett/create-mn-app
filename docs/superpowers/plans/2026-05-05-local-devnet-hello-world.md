# Local-devnet hello-world template + E2E CI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert the bundled `hello-world` template to a fully local, fully pinned, fully non-interactive devnet workflow, and verify the entire scaffolding+setup+deploy pipeline end-to-end in CI on Linux and macOS.

**Architecture:** Three-service Docker Compose devnet (`node` + `indexer` + `proof-server`) shipped inside the template. Deploy uses the well-known dev genesis seed directly. A new `scripts/e2e-check.ts` reconnects to the deployed contract via `findDeployedContract` and reads its ledger state. Three GitHub workflows (PR/push, nightly, post-publish) call a shared reusable workflow that scaffolds, sets up, deploys, and runs the e2e check.

**Tech Stack:** TypeScript / Node 22 / Vitest / Mustache (template engine, already wired up) / Docker Compose v2 / GitHub Actions (`workflow_call`, `workflow_run`, matrix) / Colima (macOS Docker) / `@midnight-ntwrk/*` SDK / Compact compiler.

**Source spec:** `~/.claude/plans/i-want-to-update-enumerated-pebble.md` — decisions D1–D10, file-by-file changes, and verification steps are locked there. Treat that file as the source of truth if anything in this plan is ambiguous.

---

## File map

Lock decomposition before tasking. Files this plan touches:

**Template (rendered into the user's project):**
- `templates/hello-world/docker-compose.yml.template` — modified: 3-service stack
- `templates/hello-world/package.json.template` — modified: scripts, exact-pin every dep
- `templates/hello-world/_gitignore` — modified: drop `.midnight-seed`
- `templates/hello-world/README.md.template` — modified: rewrite setup section
- `templates/hello-world/src/deploy.ts.template` — modified: non-interactive, `localDevnet` config, genesis seed
- `templates/hello-world/src/cli.ts.template` — modified: same `localDevnet` config, network label
- `templates/hello-world/src/check-balance.ts.template` — modified: same `localDevnet` config, drop faucet URL
- `templates/hello-world/scripts/e2e-check.ts.template` — **new**: smoke + read-back

**Scaffolder (the create-mn-app code itself):**
- `src/__tests__/template-manager.test.ts` — modified: file-presence assertion includes new files
- `src/test.ts` — modified: `requiredFiles` list

**Repo root:**
- `.compact-version` — **new**: single source of truth for compiler version
- `README.md` — modified: setup description + project tree

**CI:**
- `.github/workflows/_e2e-job.yml` — **new**: reusable
- `.github/workflows/e2e.yml` — **new**: PR + push + dispatch
- `.github/workflows/e2e-nightly.yml` — **new**: cron
- `.github/workflows/e2e-postpublish.yml` — **new**: workflow_run

**Phase numbering matches the spec's "Implementation order"** (phases 0–7; the spec has 7 phases plus a Phase 0 added here for value verification).

---

## Phase 0: Verify external values

Per the spec's "Values to verify during implementation" section. **None of these may be guessed** — the system has flagged training data on Midnight as unreliable. Each value is captured here and used in later phases.

### Task 0.1: Verify Compact compiler version

- [ ] **Step 1: Check latest available**

```bash
compact check
```

Capture the version string (e.g., `0.31.0`). Record as `COMPACT_VERSION`.

- [ ] **Step 2: Verify it can be installed via the official installer**

```bash
compact self check
```

Confirm the developer-tools version is current. If a newer dev-tools version is offered, follow `/midnight-tooling:compact-cli` to upgrade before continuing — old dev tools may not be able to install pinned compiler versions.

### Task 0.2: Verify pinned npm package versions

- [ ] **Step 1: Look up current latest for every `@midnight-ntwrk/*` dep in the template**

Read `templates/hello-world/package.json.template` to get the dependency list. For each, run:

```bash
npm view @midnight-ntwrk/compact-js version
npm view @midnight-ntwrk/compact-runtime version
npm view @midnight-ntwrk/ledger-v8 version
npm view @midnight-ntwrk/midnight-js-contracts version
npm view @midnight-ntwrk/midnight-js-http-client-proof-provider version
npm view @midnight-ntwrk/midnight-js-indexer-public-data-provider version
npm view @midnight-ntwrk/midnight-js-level-private-state-provider version
npm view @midnight-ntwrk/midnight-js-network-id version
npm view @midnight-ntwrk/midnight-js-node-zk-config-provider version
npm view @midnight-ntwrk/midnight-js-types version
npm view @midnight-ntwrk/midnight-js-utils version
npm view @midnight-ntwrk/wallet-sdk-dust-wallet version
npm view @midnight-ntwrk/wallet-sdk-facade version
npm view @midnight-ntwrk/wallet-sdk-hd version
npm view @midnight-ntwrk/wallet-sdk-shielded version
npm view @midnight-ntwrk/wallet-sdk-unshielded-wallet version
```

**Important:** Verify these versions are mutually compatible. The template's existing `package.json` has a known-working set; if the latest versions of each are NOT all in the same compatibility window, prefer keeping the existing pinned set rather than upgrading individual packages. The user's auto-memory says: "never downgrade based on memory, always verify."

Record the resolved set as `SDK_VERSIONS`.

- [ ] **Step 2: Verify non-Midnight deps**

```bash
npm view rxjs version
npm view ws version
npm view tsx version
npm view typescript version
npm view @types/node version
npm view @types/ws version
```

These are looser; pick the latest stable that the existing template lockfile is compatible with. Record as `OTHER_VERSIONS`.

### Task 0.3: Verify Docker image tags for the local devnet stack

- [ ] **Step 1: Check the devnet skill's reference template for current known-good tags**

```bash
cat ~/.claude/plugins/cache/midnight-expert/midnight-tooling/0.3.1/skills/devnet/references/version-resolution.md
cat ~/.claude/plugins/cache/midnight-expert/midnight-tooling/0.3.1/skills/devnet/references/docker-setup.md
```

Look for explicit tags or a resolution algorithm.

- [ ] **Step 2: Cross-check against Docker Hub**

```bash
# List recent tags for each image; pick a stable (non-rc) tag
docker run --rm regclient/regctl image tag ls midnightntwrk/midnight-node 2>/dev/null | head -20
docker run --rm regclient/regctl image tag ls midnightntwrk/indexer-standalone 2>/dev/null | head -20
docker run --rm regclient/regctl image tag ls midnightntwrk/proof-server 2>/dev/null | head -20
```

If `regctl` is unavailable, the equivalent via Docker Hub API:

```bash
curl -s "https://hub.docker.com/v2/repositories/midnightntwrk/midnight-node/tags?page_size=20" | jq -r '.results[].name'
curl -s "https://hub.docker.com/v2/repositories/midnightntwrk/indexer-standalone/tags?page_size=20" | jq -r '.results[].name'
curl -s "https://hub.docker.com/v2/repositories/midnightntwrk/proof-server/tags?page_size=20" | jq -r '.results[].name'
```

The current template uses `proof-server:7.0.0` — that's the existing baseline. Record the chosen trio as `NODE_IMAGE_TAG`, `INDEXER_IMAGE_TAG`, `PROOF_SERVER_IMAGE_TAG`. **Prefer the tags already in the devnet skill's `templates/devnet.yml`** if its variable substitution defaults reveal a known-tested set; otherwise pick the latest stable trio from the same release window.

- [ ] **Step 3: Smoke-pull all three locally**

```bash
docker pull midnightntwrk/midnight-node:$NODE_IMAGE_TAG
docker pull midnightntwrk/indexer-standalone:$INDEXER_IMAGE_TAG
docker pull midnightntwrk/proof-server:$PROOF_SERVER_IMAGE_TAG
```

All three must succeed.

### Task 0.4: Verify the SDK network-id call

- [ ] **Step 1: Confirm `setNetworkId('undeployed')` is a valid argument**

```bash
mkdir -p /tmp/midnight-network-id-check && cd /tmp/midnight-network-id-check
npm init -y >/dev/null
npm install @midnight-ntwrk/midnight-js-network-id@$(npm view @midnight-ntwrk/midnight-js-network-id version) >/dev/null 2>&1
node -e "
  const m = require('@midnight-ntwrk/midnight-js-network-id');
  m.setNetworkId('undeployed');
  console.log('OK:', m.getNetworkId());
"
```

Expected: `OK: undeployed`. If the call throws, inspect the package's exported types:

```bash
cat node_modules/@midnight-ntwrk/midnight-js-network-id/dist/*.d.ts | grep -E '(NetworkId|setNetworkId)'
```

The valid set of network IDs will be visible in the type signature. If `'undeployed'` is not accepted, find the correct local-network identifier and use that throughout. Record as `LOCAL_NETWORK_ID`.

- [ ] **Step 2: Cleanup**

```bash
cd ~ && rm -rf /tmp/midnight-network-id-check
```

### Task 0.5: Verify indexer GraphQL endpoint path on local devnet

- [ ] **Step 1: Boot just the devnet stack to interrogate it**

Use the devnet skill's compose file directly:

```bash
mkdir -p /tmp/devnet-probe && cd /tmp/devnet-probe
cp ~/.claude/plugins/cache/midnight-expert/midnight-tooling/0.3.1/skills/devnet/templates/devnet.yml ./compose.yml
# Substitute the version placeholders we resolved in Task 0.3
sed -i.bak \
  -e "s/{{NODE_VERSION}}/$NODE_IMAGE_TAG/" \
  -e "s/{{INDEXER_VERSION}}/$INDEXER_IMAGE_TAG/" \
  -e "s/{{PROOF_SERVER_VERSION}}/$PROOF_SERVER_IMAGE_TAG/" \
  compose.yml
docker compose up -d --wait
```

- [ ] **Step 2: Confirm the GraphQL endpoint path**

```bash
# The standard endpoint per the existing template's preprod URL is /api/v3/graphql
curl -s -o /dev/null -w "%{http_code}\n" -X POST -H "Content-Type: application/json" \
  -d '{"query":"{ __typename }"}' \
  http://127.0.0.1:8088/api/v3/graphql
```

Expected: `200`. If `404`, try `/graphql` or `/api/v1/graphql` — record whatever path returns 200 as `INDEXER_PATH`.

- [ ] **Step 3: Tear down**

```bash
docker compose down -v
cd ~ && rm -rf /tmp/devnet-probe
```

### Task 0.6: Verify the macOS Docker setup action

- [ ] **Step 1: Pin a maintained Colima-installer action**

Search the GitHub Marketplace for currently-maintained options. Two candidates:

```bash
# Option A: douglascamata/setup-docker-macos-action — check last release date
curl -s https://api.github.com/repos/douglascamata/setup-docker-macos-action/releases/latest | jq -r '.tag_name + " — " + .published_at'

# Option B: install colima directly via brew (no third-party action dependency)
# Documented as: brew install docker docker-compose colima && colima start
```

Prefer **Option B (brew install)** if Option A's last release is >12 months old or the repo is archived. Record the chosen approach as `MACOS_DOCKER_SETUP`.

### Task 0.7: Capture all values

- [ ] **Step 1: Write a values record**

Create `/tmp/local-devnet-values.txt` with all captured values. This file is reference-only and not committed:

```
COMPACT_VERSION=<from 0.1>
SDK_VERSIONS:
  @midnight-ntwrk/compact-js=<version>
  ... (full list)
OTHER_VERSIONS:
  rxjs=<version>
  ... (full list)
NODE_IMAGE_TAG=<from 0.3>
INDEXER_IMAGE_TAG=<from 0.3>
PROOF_SERVER_IMAGE_TAG=<from 0.3>
LOCAL_NETWORK_ID=<from 0.4>
INDEXER_PATH=<from 0.5>
MACOS_DOCKER_SETUP=<A or B from 0.6>
```

These values are used verbatim in subsequent phases. Do not commit this file.

- [ ] **Step 2: Phase 0 has no commit**

Phase 0 is information-gathering only. No code changes, nothing to commit. Move to Phase 1.

---

## Phase 1: Refactor `deploy.ts.template` to non-interactive (still Preprod)

Goal: separate the "remove prompts + extract config object" change from the "switch networks" change. After Phase 1, scaffolding still produces a Preprod-targeted app, but with no prompts and a clean `localDevnet`-shaped config object ready to flip.

### Task 1.1: Write the rewritten `deploy.ts.template`

**Files:**
- Modify: `templates/hello-world/src/deploy.ts.template` (full rewrite of the body; imports unchanged)

- [ ] **Step 1: Read the current file to confirm imports**

```bash
sed -n '1,32p' templates/hello-world/src/deploy.ts.template
```

Imports stay as-is. Body is rewritten.

- [ ] **Step 2: Replace the file contents**

Write the full new file content. (Imports section is the same as the current file lines 1–32; the body below replaces lines 33-end.)

```typescript
/**
 * Deploy {{projectName}} contract to Midnight Preprod network.
 *
 * Non-interactive: scaffold → npm run setup runs straight through.
 * No readline prompts, no .midnight-seed file.
 */
import * as fs from 'node:fs';
import * as path from 'node:path';
import { fileURLToPath, pathToFileURL } from 'node:url';
import { WebSocket } from 'ws';
import * as Rx from 'rxjs';
import { Buffer } from 'buffer';

// Midnight SDK imports
import { deployContract } from '@midnight-ntwrk/midnight-js-contracts';
import { httpClientProofProvider } from '@midnight-ntwrk/midnight-js-http-client-proof-provider';
import { indexerPublicDataProvider } from '@midnight-ntwrk/midnight-js-indexer-public-data-provider';
import { levelPrivateStateProvider } from '@midnight-ntwrk/midnight-js-level-private-state-provider';
import { NodeZkConfigProvider } from '@midnight-ntwrk/midnight-js-node-zk-config-provider';
import { setNetworkId, getNetworkId } from '@midnight-ntwrk/midnight-js-network-id';
import { toHex } from '@midnight-ntwrk/midnight-js-utils';
import * as ledger from '@midnight-ntwrk/ledger-v8';
import { unshieldedToken } from '@midnight-ntwrk/ledger-v8';
import { WalletFacade } from '@midnight-ntwrk/wallet-sdk-facade';
import { DustWallet } from '@midnight-ntwrk/wallet-sdk-dust-wallet';
import { HDWallet, Roles, generateRandomSeed } from '@midnight-ntwrk/wallet-sdk-hd';
import { ShieldedWallet } from '@midnight-ntwrk/wallet-sdk-shielded';
import {
  createKeystore,
  InMemoryTransactionHistoryStorage,
  PublicKey,
  UnshieldedWallet,
} from '@midnight-ntwrk/wallet-sdk-unshielded-wallet';
import { CompiledContract } from '@midnight-ntwrk/compact-js';

// @ts-expect-error Required for wallet sync
globalThis.WebSocket = WebSocket;

// ─── Network configuration ─────────────────────────────────────────────────────
//
// Single source of truth for everything the wallet and providers need to point at.
// To add another deploy target later (e.g. Preprod), define a sibling object with
// the same shape and select between them. Don't add the selector until it's needed.

const localDevnet = {
  networkId: 'preprod' as const, // PHASE 1: keep Preprod for now; Phase 2 flips this.
  indexer: 'https://indexer.preprod.midnight.network/api/v3/graphql',
  indexerWS: 'wss://indexer.preprod.midnight.network/api/v3/graphql/ws',
  node: 'https://rpc.preprod.midnight.network',
  proofServer: 'http://127.0.0.1:6300',
  faucetUrl: 'https://faucet.preprod.midnight.network/',
};

setNetworkId(localDevnet.networkId);

// ─── Proof server health check ─────────────────────────────────────────────────

async function waitForProofServer(maxAttempts = 30, delayMs = 2000): Promise<boolean> {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      await fetch(localDevnet.proofServer, {
        method: 'GET',
        signal: AbortSignal.timeout(3000),
      });
      return true;
    } catch (err: any) {
      const errMsg = err?.cause?.code || err?.code || '';
      if (errMsg !== 'ECONNREFUSED' && errMsg !== 'UND_ERR_CONNECT_TIMEOUT') {
        return true;
      }
    }
    if (attempt < maxAttempts) {
      process.stdout.write(`\r  Waiting for proof server... (${attempt}/${maxAttempts})   `);
      await new Promise((r) => setTimeout(r, delayMs));
    }
  }
  return false;
}

// ─── Compiled contract loading ─────────────────────────────────────────────────

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const zkConfigPath = path.resolve(__dirname, '..', 'contracts', 'managed', 'hello-world');
const contractPath = path.join(zkConfigPath, 'contract', 'index.js');

if (!fs.existsSync(contractPath)) {
  console.error('\n❌ Contract not compiled! Run: npm run compile\n');
  process.exit(1);
}

const HelloWorld = await import(pathToFileURL(contractPath).href);

const compiledContract = CompiledContract.make('hello-world', HelloWorld.Contract).pipe(
  CompiledContract.withVacantWitnesses,
  CompiledContract.withCompiledFileAssets(zkConfigPath),
);

// ─── Wallet helpers ────────────────────────────────────────────────────────────

function deriveKeys(seed: string) {
  const hdWallet = HDWallet.fromSeed(Buffer.from(seed, 'hex'));
  if (hdWallet.type !== 'seedOk') throw new Error('Invalid seed');
  const result = hdWallet.hdWallet
    .selectAccount(0)
    .selectRoles([Roles.Zswap, Roles.NightExternal, Roles.Dust])
    .deriveKeysAt(0);
  if (result.type !== 'keysDerived') throw new Error('Key derivation failed');
  hdWallet.hdWallet.clear();
  return result.keys;
}

async function createWallet(seed: string) {
  const keys = deriveKeys(seed);
  const networkId = getNetworkId();
  const shieldedSecretKeys = ledger.ZswapSecretKeys.fromSeed(keys[Roles.Zswap]);
  const dustSecretKey = ledger.DustSecretKey.fromSeed(keys[Roles.Dust]);
  const unshieldedKeystore = createKeystore(keys[Roles.NightExternal], networkId);

  const walletConfig = {
    networkId,
    indexerClientConnection: { indexerHttpUrl: localDevnet.indexer, indexerWsUrl: localDevnet.indexerWS },
    provingServerUrl: new URL(localDevnet.proofServer),
    relayURL: new URL(localDevnet.node.replace(/^http/, 'ws')),
    txHistoryStorage: new InMemoryTransactionHistoryStorage(),
    costParameters: { additionalFeeOverhead: 300_000_000_000_000n, feeBlocksMargin: 5 },
  };

  const wallet = await WalletFacade.init({
    configuration: walletConfig,
    shielded: async (config) => ShieldedWallet(config).startWithSecretKeys(shieldedSecretKeys),
    unshielded: async (config) =>
      UnshieldedWallet(config).startWithPublicKey(PublicKey.fromKeyStore(unshieldedKeystore)),
    dust: async (config) =>
      DustWallet(config).startWithSecretKey(dustSecretKey, ledger.LedgerParameters.initialParameters().dust),
  });

  await wallet.start(shieldedSecretKeys, dustSecretKey);

  return { wallet, shieldedSecretKeys, dustSecretKey, unshieldedKeystore };
}

async function createProviders(walletCtx: ReturnType<typeof createWallet> extends Promise<infer T> ? T : never) {
  const privateStatePassword = process.env.PRIVATE_STATE_PASSWORD?.trim() || 'development';
  const state = await walletCtx.wallet.waitForSyncedState();

  const walletProvider = {
    getCoinPublicKey: () => state.shielded.coinPublicKey.toHexString(),
    getEncryptionPublicKey: () => state.shielded.encryptionPublicKey.toHexString(),
    async balanceTx(tx: any, ttl?: Date) {
      const recipe = await walletCtx.wallet.balanceUnboundTransaction(
        tx,
        { shieldedSecretKeys: walletCtx.shieldedSecretKeys, dustSecretKey: walletCtx.dustSecretKey },
        { ttl: ttl ?? new Date(Date.now() + 30 * 60 * 1000) },
      );
      const signedRecipe = await walletCtx.wallet.signRecipe(recipe, (payload) =>
        walletCtx.unshieldedKeystore.signData(payload),
      );
      return walletCtx.wallet.finalizeRecipe(signedRecipe);
    },
    submitTx: (tx: any) => walletCtx.wallet.submitTransaction(tx) as any,
  };

  const zkConfigProvider = new NodeZkConfigProvider(zkConfigPath);
  const accountId = walletCtx.unshieldedKeystore.getBech32Address().toString();

  return {
    privateStateProvider: levelPrivateStateProvider({
      privateStateStoreName: 'hello-world-state',
      accountId,
      privateStoragePasswordProvider: () => privateStatePassword,
    }),
    publicDataProvider: indexerPublicDataProvider(localDevnet.indexer, localDevnet.indexerWS),
    zkConfigProvider,
    proofProvider: httpClientProofProvider(localDevnet.proofServer, zkConfigProvider),
    walletProvider,
    midnightProvider: walletProvider,
  };
}

// ─── Main ──────────────────────────────────────────────────────────────────────

async function main() {
  console.log('\n╔══════════════════════════════════════════════════════════════╗');
  console.log('║           Deploy {{projectName}}                              ║');
  console.log('╚══════════════════════════════════════════════════════════════╝\n');

  // Phase 1: still uses Preprod faucet flow. Phase 2 swaps the seed and removes
  // the faucet wait. We keep `generateRandomSeed` here so a fresh wallet is used
  // each run; deployment.json keeps a record of contract address only — no seed.
  const seed = toHex(Buffer.from(generateRandomSeed()));

  console.log('─── Wallet setup ───────────────────────────────────────────────\n');
  console.log('  Creating wallet...');
  const walletCtx = await createWallet(seed);

  console.log('  Syncing with network...');
  console.log('  ℹ  This may take several minutes depending on network size.');
  console.log('     RPC disconnection messages during sync are normal and can be safely ignored.\n');
  const syncStart = Date.now();
  const syncInterval = setInterval(() => {
    const elapsed = Math.round((Date.now() - syncStart) / 1000);
    process.stdout.write(`\r  ⏳ Still syncing... (${elapsed}s elapsed)   `);
  }, 5000);
  const state = await walletCtx.wallet.waitForSyncedState();
  clearInterval(syncInterval);
  process.stdout.write('\r  ✓ Synced with network.                                      \n');

  const address = walletCtx.unshieldedKeystore.getBech32Address();
  let balance = state.unshielded.balances[unshieldedToken().raw] ?? 0n;
  console.log(`\n  Wallet Address: ${address}`);
  console.log(`  Balance: ${balance.toLocaleString()} tNight\n`);

  // Fund via faucet (Phase 1 only — Phase 2 removes this entirely).
  if (balance === 0n) {
    console.log('─── Fund Your Wallet ───────────────────────────────────────────\n');
    console.log(`  Visit: ${localDevnet.faucetUrl}`);
    console.log(`  Address: ${address}\n`);
    console.log('  Waiting for funds...');

    await Rx.firstValueFrom(
      walletCtx.wallet.state().pipe(
        Rx.throttleTime(10000),
        Rx.filter((s) => s.isSynced),
        Rx.map((s) => s.unshielded.balances[unshieldedToken().raw] ?? 0n),
        Rx.filter((b) => b > 0n),
      ),
    );
    console.log('  Funds received!\n');
  }

  // Register for DUST.
  console.log('─── DUST Token Setup ───────────────────────────────────────────\n');
  const dustState = await Rx.firstValueFrom(walletCtx.wallet.state().pipe(Rx.filter((s) => s.isSynced)));

  if (dustState.dust.balance(new Date()) === 0n) {
    const nightUtxos = dustState.unshielded.availableCoins.filter(
      (c: any) => !c.meta?.registeredForDustGeneration,
    );
    if (nightUtxos.length > 0) {
      console.log('  Registering for DUST generation...');
      const recipe = await walletCtx.wallet.registerNightUtxosForDustGeneration(
        nightUtxos,
        walletCtx.unshieldedKeystore.getPublicKey(),
        (payload) => walletCtx.unshieldedKeystore.signData(payload),
      );
      const signedRecipe = await walletCtx.wallet.signRecipe(recipe, (payload) =>
        walletCtx.unshieldedKeystore.signData(payload),
      );
      await walletCtx.wallet.submitTransaction(await walletCtx.wallet.finalizeRecipe(signedRecipe));
    }

    console.log('  Waiting for DUST tokens...');
    await Rx.firstValueFrom(
      walletCtx.wallet.state().pipe(
        Rx.throttleTime(5000),
        Rx.filter((s) => s.isSynced),
        Rx.filter((s) => s.dust.balance(new Date()) > 0n),
      ),
    );
  }
  console.log('  DUST tokens ready!\n');

  // Deploy.
  console.log('─── Deploy Contract ────────────────────────────────────────────\n');

  console.log('  Checking proof server...');
  const proofServerReady = await waitForProofServer();
  if (!proofServerReady) {
    console.log('\n  ❌ Proof server not responding. Run: docker compose up -d\n');
    fs.writeFileSync(
      'deployment.json',
      JSON.stringify({ address, network: localDevnet.networkId, status: 'proof_server_unavailable' }, null, 2),
    );
    await walletCtx.wallet.stop();
    process.exit(1);
  }
  process.stdout.write('\r  Proof server ready!                    \n');

  console.log('  Setting up providers...');
  const providers = await createProviders(walletCtx);

  console.log('  Deploying contract...\n');

  const MAX_RETRIES = 8;
  const RETRY_DELAY_MS = 15000;
  let deployed: Awaited<ReturnType<typeof deployContract>> | undefined;

  for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
    try {
      deployed = await deployContract(providers, {
        compiledContract: compiledContract as any,
        args: [],
      });
      break;
    } catch (err: any) {
      const errMsg = err?.message || err?.toString() || '';
      const errCause = err?.cause?.message || err?.cause?.toString() || '';
      const fullError = `${errMsg} ${errCause}`;

      if (
        fullError.includes('Failed to connect to Proof Server') ||
        fullError.includes('Failed to prove') ||
        fullError.includes('127.0.0.1:6300')
      ) {
        console.log('  ❌ Proof server error during deploy. Run: docker compose up -d\n');
        fs.writeFileSync(
          'deployment.json',
          JSON.stringify({ address, network: localDevnet.networkId, status: 'proof_server_error' }, null, 2),
        );
        await walletCtx.wallet.stop();
        process.exit(1);
      }

      if (fullError.includes('Not enough Dust')) {
        const currentState = await walletCtx.wallet.waitForSyncedState();
        const dustBalance = currentState.dust.balance(new Date());
        if (attempt < MAX_RETRIES) {
          console.log(`  ⏳ DUST balance: ${dustBalance.toLocaleString()} (attempt ${attempt}/${MAX_RETRIES})`);
          for (let i = RETRY_DELAY_MS / 1000; i > 0; i -= 5) {
            process.stdout.write(`\r     Retrying in ${i}s...   `);
            await new Promise((r) => setTimeout(r, 5000));
          }
          process.stdout.write('\r                              \r\n');
        } else {
          console.log(`  ❌ Not enough DUST after ${MAX_RETRIES} retries (current: ${dustBalance.toLocaleString()})`);
          fs.writeFileSync(
            'deployment.json',
            JSON.stringify(
              { address, network: localDevnet.networkId, status: 'pending_dust', lastAttempt: new Date().toISOString() },
              null,
              2,
            ),
          );
          await walletCtx.wallet.stop();
          process.exit(1);
        }
      } else {
        throw err;
      }
    }
  }

  if (!deployed) throw new Error('Deployment failed after all retries');

  const contractAddress = deployed.deployTxData.public.contractAddress;
  console.log('  ✅ Contract deployed successfully!\n');
  console.log(`  Contract Address: ${contractAddress}\n`);

  fs.writeFileSync(
    'deployment.json',
    JSON.stringify(
      { contractAddress, network: localDevnet.networkId, deployedAt: new Date().toISOString() },
      null,
      2,
    ),
  );
  console.log('  Saved to deployment.json\n');

  await walletCtx.wallet.stop();
  console.log('─── Deployment complete ────────────────────────────────────────\n');
  console.log('  Next: npm run cli\n');
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

Use the Write tool to overwrite `templates/hello-world/src/deploy.ts.template` with the imports section (lines 1–32 of the existing file) followed by the body above starting from the docstring `/** Deploy ... */`.

- [ ] **Step 3: Verify the file syntactically renders as a Mustache template**

The only Mustache placeholders in the new file are `{{projectName}}` (used twice — in the docstring and in the `console.log` banner). This matches existing template conventions.

```bash
grep -n '{{' templates/hello-world/src/deploy.ts.template | head
```

Expected: only `{{projectName}}` references; no leftover `{{` from elsewhere.

### Task 1.2: Verify scaffolder still works

- [ ] **Step 1: Build the scaffolder**

```bash
npm run build
```

Expected: clean build, no TypeScript errors.

- [ ] **Step 2: Run unit tests**

```bash
npm run test
```

Expected: all pass. The `template-manager.test.ts` tests a hello-world scaffold and the rendered files exist; the file paths it checks haven't changed yet, so this should pass without modification.

- [ ] **Step 3: Run the smoke test**

```bash
npm run test:smoke
```

Expected: pass.

- [ ] **Step 4: Manually scaffold and type-check the rendered template**

```bash
rm -rf /tmp/phase1-sandbox
node bin/create-midnight-app.js /tmp/phase1-sandbox -y -t hello-world --skip-install
cd /tmp/phase1-sandbox
npm install --no-audit --prefer-offline
npx tsc --noEmit
cd -
```

Expected: `npx tsc --noEmit` passes (no type errors in the rendered `deploy.ts`). Note: we use `--skip-install` then a manual install to avoid running the full setup at this stage.

### Task 1.3: Commit Phase 1

- [ ] **Step 1: Stage and commit**

```bash
git add templates/hello-world/src/deploy.ts.template
git commit -m "$(cat <<'EOF'
refactor(hello-world): make deploy.ts non-interactive, extract localDevnet config

Strip readline prompts and .midnight-seed persistence; introduce a single
localDevnet config object (still pointed at Preprod) so Phase 2 can flip
endpoints without restructuring the deploy flow. No behavioural change for
end users yet beyond the loss of interactive prompts (the random-seed +
faucet path runs straight through).

Refs: docs/superpowers/plans/2026-05-05-local-devnet-hello-world.md (Phase 1)

Co-Authored-By: Claude Code <noreply@anthropic.com>
EOF
)"
```

---

## Phase 2: Local devnet stack + flip `deploy.ts.template` to local

Goal: ship the three-service compose file, swap the deploy script to use the genesis seed + local endpoints, and verify a real contract address comes out the other end.

### Task 2.1: Write the new `docker-compose.yml.template`

**Files:**
- Modify: `templates/hello-world/docker-compose.yml.template`

- [ ] **Step 1: Replace the file**

Use the values from Phase 0 (`NODE_IMAGE_TAG`, `INDEXER_IMAGE_TAG`, `PROOF_SERVER_IMAGE_TAG`). Write:

```yaml
# Local Midnight devnet for {{projectName}}
# LOCAL DEVELOPMENT ONLY — do not expose these ports to a public network.
# Tear down with: docker compose down -v
name: {{kebabName}}-devnet

services:
  node:
    image: midnightntwrk/midnight-node:<NODE_IMAGE_TAG>
    container_name: {{kebabName}}-node
    ports:
      - "9944:9944"
    environment:
      CFG_PRESET: "dev"
      SIDECHAIN_BLOCK_BENEFICIARY: "04bcf7ad3be7a5c790460be82a713af570f22e0f801f6659ab8e84a52be6969e"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9944/health"]
      interval: 2s
      timeout: 5s
      retries: 30
      start_period: 20s

  indexer:
    image: midnightntwrk/indexer-standalone:<INDEXER_IMAGE_TAG>
    container_name: {{kebabName}}-indexer
    ports:
      - "8088:8088"
    environment:
      RUST_LOG: "indexer=info,chain_indexer=info,indexer_api=info,wallet_indexer=info,indexer_common=info,fastrace_opentelemetry=off,info"
      APP__APPLICATION__NETWORK_ID: "undeployed"
      APP__INFRA__NODE__URL: "ws://node:9944"
      APP__INFRA__STORAGE__PASSWORD: "indexer"
      APP__INFRA__PUB_SUB__PASSWORD: "indexer"
      APP__INFRA__LEDGER_STATE_STORAGE__PASSWORD: "indexer"
      APP__INFRA__SECRET: "303132333435363738393031323334353637383930313233343536373839303132"
    healthcheck:
      test: ["CMD-SHELL", "cat /var/run/indexer-standalone/running"]
      interval: 10s
      timeout: 5s
      retries: 30
      start_period: 10s
    depends_on:
      node:
        condition: service_healthy

  proof-server:
    image: midnightntwrk/proof-server:<PROOF_SERVER_IMAGE_TAG>
    container_name: {{kebabName}}-proof-server
    command: ["midnight-proof-server -v"]
    ports:
      - "6300:6300"
    environment:
      RUST_BACKTRACE: "full"
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:6300 >/dev/null 2>&1 || exit 1"]
      interval: 5s
      timeout: 5s
      retries: 30
      start_period: 10s
```

Substitute `<NODE_IMAGE_TAG>`, `<INDEXER_IMAGE_TAG>`, `<PROOF_SERVER_IMAGE_TAG>` with the literal values from Phase 0.7. **Do not leave angle-bracket placeholders in the file.**

If `LOCAL_NETWORK_ID` from Task 0.4 is something other than `undeployed`, change `APP__APPLICATION__NETWORK_ID` and the `setNetworkId(...)` call in deploy.ts to match.

- [ ] **Step 2: Confirm Mustache placeholders**

```bash
grep -n '{{' templates/hello-world/docker-compose.yml.template
```

Expected: only `{{projectName}}` and `{{kebabName}}` — both are already injected by `template-manager.ts`.

### Task 2.2: Flip `deploy.ts.template` to local devnet

**Files:**
- Modify: `templates/hello-world/src/deploy.ts.template`

- [ ] **Step 1: Update the `localDevnet` config object and seed**

Edit the block defined in Task 1.1. Replace the `localDevnet` object and add the genesis seed constant. The new shape (between the imports and the `setNetworkId` call):

```typescript
// ─── Network configuration ─────────────────────────────────────────────────────
//
// Single source of truth for everything the wallet and providers need to point at.
// To add another deploy target later (e.g. Preprod), define a sibling object with
// the same shape and select between them. Don't add the selector until it's needed.

const localDevnet = {
  networkId: '<LOCAL_NETWORK_ID>' as const,
  indexer: 'http://127.0.0.1:8088<INDEXER_PATH>',
  indexerWS: 'ws://127.0.0.1:8088<INDEXER_PATH>/ws',
  node: 'ws://127.0.0.1:9944',
  proofServer: 'http://127.0.0.1:6300',
};

setNetworkId(localDevnet.networkId);

// ─── Genesis seed ──────────────────────────────────────────────────────────────
//
// LOCAL DEVNET ONLY.
// The local devnet's `dev` preset pre-mints NIGHT to the wallet derived from
// this well-known seed. Anyone running this devnet has full access to the funds
// at this seed — never reuse it on Preprod, mainnet, or any environment that
// handles real value.
const GENESIS_SEED = '0000000000000000000000000000000000000000000000000000000000000001';
```

Substitute `<LOCAL_NETWORK_ID>` and `<INDEXER_PATH>` with the literal values from Phase 0.

Also remove `faucetUrl` from the config and remove the `generateRandomSeed` import use.

- [ ] **Step 2: Update the imports**

Remove `generateRandomSeed` from the `@midnight-ntwrk/wallet-sdk-hd` import (it's no longer used). Remove `toHex` from `@midnight-ntwrk/midnight-js-utils` (no longer used). The imports should now be:

```typescript
import { HDWallet, Roles } from '@midnight-ntwrk/wallet-sdk-hd';
```

(Drop the `@midnight-ntwrk/midnight-js-utils` import line entirely if `toHex` was the only thing it imported.)

- [ ] **Step 3: Drop `waitForProofServer`**

Delete the entire `waitForProofServer` function and the `proofServerReady` check. Docker `--wait` covers it. If the function is referenced anywhere else, remove those references.

Replace the "Checking proof server..." block with nothing — delete those lines.

- [ ] **Step 4: Replace the `seed = ...` line in `main`**

Replace:
```typescript
const seed = toHex(Buffer.from(generateRandomSeed()));
```
with:
```typescript
const seed = GENESIS_SEED;
```

- [ ] **Step 5: Delete the faucet-wait block**

Remove the entire `if (balance === 0n) { … "Funds received" … }` block. NIGHT is pre-minted on the genesis-seed wallet; if it's missing, the devnet didn't boot correctly and we want a hard failure, not a silent wait.

After removal, immediately after the synced-balance log, insert:

```typescript
if (balance === 0n) {
  console.error(
    '\n❌ Genesis-seed wallet has zero NIGHT. The devnet preset may not have minted to it.\n' +
      '   Check `docker compose ps` and `docker compose logs node`. Then `docker compose down -v` and retry.\n',
  );
  await walletCtx.wallet.stop();
  process.exit(1);
}
```

- [ ] **Step 6: Update banner and final messages**

The deploy banner becomes:

```typescript
console.log('║           Deploy {{projectName}} to local devnet              ║');
```

The deployment.json `network` field stays as `localDevnet.networkId` (which is now the local value).

- [ ] **Step 7: Update the setup script in `package.json.template`**

Edit `templates/hello-world/package.json.template`. Replace the `"setup"` line:

```json
"setup": "docker compose up -d --wait && npm run compile && npm run deploy",
```

(`--wait` already implicitly handles the proof server health check. The change vs current is the new compose stack having more services, all of which `--wait` will block on.)

### Task 2.3: Manual end-to-end verification

- [ ] **Step 1: Build, scaffold, run setup**

```bash
npm run build
rm -rf /tmp/phase2-sandbox
node bin/create-midnight-app.js /tmp/phase2-sandbox -y -t hello-world
cd /tmp/phase2-sandbox
npm run setup
```

Expected: compose comes up healthy on all three services; compile succeeds; deploy outputs `✅ Contract deployed successfully!` and writes `deployment.json` with `contractAddress` and `network`.

- [ ] **Step 2: Inspect the artifact**

```bash
jq -e '.contractAddress | type == "string" and length > 0' deployment.json
jq -r '.network' deployment.json
```

Expected: jq exit 0; network value matches `LOCAL_NETWORK_ID`.

- [ ] **Step 3: Tear down**

```bash
docker compose down -v
cd -
```

Expected: clean shutdown, no residual volumes.

### Task 2.4: Commit Phase 2

- [ ] **Step 1: Stage and commit**

```bash
git add templates/hello-world/docker-compose.yml.template \
        templates/hello-world/src/deploy.ts.template \
        templates/hello-world/package.json.template
git commit -m "$(cat <<'EOF'
feat(hello-world): switch to bundled local devnet for setup + deploy

docker-compose.yml.template now ships node + indexer + proof-server, all
pinned to exact image tags. deploy.ts uses the dev preset's genesis seed
directly with a LOCAL DEVNET ONLY warning; faucet-wait and proof-server
polling are gone (Docker `--wait` covers the latter). `npm run setup`
runs straight through with no prompts and produces a deployed contract
on every clean machine.

Refs: docs/superpowers/plans/2026-05-05-local-devnet-hello-world.md (Phase 2)

Co-Authored-By: Claude Code <noreply@anthropic.com>
EOF
)"
```

---

## Phase 3: `scripts/e2e-check.ts.template`

Goal: a single script that proves the just-deployed contract is on chain and queryable.

### Task 3.1: Create the e2e-check template

**Files:**
- Create: `templates/hello-world/scripts/e2e-check.ts.template`
- Modify: `templates/hello-world/package.json.template` — add `test:e2e` script

- [ ] **Step 1: Create the scripts directory and write the file**

Create `templates/hello-world/scripts/e2e-check.ts.template`:

```typescript
/**
 * End-to-end smoke check for {{projectName}}.
 *
 * Reads deployment.json, reconnects to the deployed contract, reads its
 * ledger state, and exits 0 on success. Used by `npm run test:e2e` and by
 * the project's CI workflows.
 */
import * as fs from 'node:fs';
import * as path from 'node:path';
import { fileURLToPath, pathToFileURL } from 'node:url';
import { WebSocket } from 'ws';
import { Buffer } from 'buffer';

import { findDeployedContract } from '@midnight-ntwrk/midnight-js-contracts';
import { httpClientProofProvider } from '@midnight-ntwrk/midnight-js-http-client-proof-provider';
import { indexerPublicDataProvider } from '@midnight-ntwrk/midnight-js-indexer-public-data-provider';
import { levelPrivateStateProvider } from '@midnight-ntwrk/midnight-js-level-private-state-provider';
import { NodeZkConfigProvider } from '@midnight-ntwrk/midnight-js-node-zk-config-provider';
import { setNetworkId, getNetworkId } from '@midnight-ntwrk/midnight-js-network-id';
import * as ledger from '@midnight-ntwrk/ledger-v8';
import { WalletFacade } from '@midnight-ntwrk/wallet-sdk-facade';
import { DustWallet } from '@midnight-ntwrk/wallet-sdk-dust-wallet';
import { HDWallet, Roles } from '@midnight-ntwrk/wallet-sdk-hd';
import { ShieldedWallet } from '@midnight-ntwrk/wallet-sdk-shielded';
import {
  createKeystore,
  InMemoryTransactionHistoryStorage,
  PublicKey,
  UnshieldedWallet,
} from '@midnight-ntwrk/wallet-sdk-unshielded-wallet';
import { CompiledContract } from '@midnight-ntwrk/compact-js';

// @ts-expect-error wallet sync
globalThis.WebSocket = WebSocket;

const localDevnet = {
  networkId: '<LOCAL_NETWORK_ID>' as const,
  indexer: 'http://127.0.0.1:8088<INDEXER_PATH>',
  indexerWS: 'ws://127.0.0.1:8088<INDEXER_PATH>/ws',
  node: 'ws://127.0.0.1:9944',
  proofServer: 'http://127.0.0.1:6300',
};

const GENESIS_SEED = '0000000000000000000000000000000000000000000000000000000000000001';

setNetworkId(localDevnet.networkId);

function fail(msg: string): never {
  console.error(`❌ e2e-check failed: ${msg}`);
  process.exit(1);
}

function isHexAddress(s: unknown): s is string {
  return typeof s === 'string' && /^[0-9a-fA-F]+$/.test(s) && s.length >= 32;
}

async function main() {
  // 1. deployment.json sanity
  if (!fs.existsSync('deployment.json')) fail('deployment.json missing — run `npm run setup` first.');
  const deployment = JSON.parse(fs.readFileSync('deployment.json', 'utf-8'));
  if (!isHexAddress(deployment.contractAddress)) {
    fail(`deployment.json missing or invalid contractAddress: ${JSON.stringify(deployment, null, 2)}`);
  }
  if (deployment.network !== localDevnet.networkId) {
    fail(`deployment.json network mismatch: expected '${localDevnet.networkId}', got '${deployment.network}'.`);
  }

  // 2. Build wallet (genesis seed) and providers
  const __dirname = path.dirname(fileURLToPath(import.meta.url));
  const zkConfigPath = path.resolve(__dirname, '..', 'contracts', 'managed', 'hello-world');
  const contractPath = path.join(zkConfigPath, 'contract', 'index.js');
  if (!fs.existsSync(contractPath)) fail('Compiled contract missing — run `npm run compile`.');
  const HelloWorld = await import(pathToFileURL(contractPath).href);
  const compiledContract = CompiledContract.make('hello-world', HelloWorld.Contract).pipe(
    CompiledContract.withVacantWitnesses,
    CompiledContract.withCompiledFileAssets(zkConfigPath),
  );

  const hd = HDWallet.fromSeed(Buffer.from(GENESIS_SEED, 'hex'));
  if (hd.type !== 'seedOk') fail('Bad seed.');
  const derived = hd.hdWallet
    .selectAccount(0)
    .selectRoles([Roles.Zswap, Roles.NightExternal, Roles.Dust])
    .deriveKeysAt(0);
  if (derived.type !== 'keysDerived') fail('Key derivation failed.');
  hd.hdWallet.clear();

  const networkId = getNetworkId();
  const shieldedSecretKeys = ledger.ZswapSecretKeys.fromSeed(derived.keys[Roles.Zswap]);
  const dustSecretKey = ledger.DustSecretKey.fromSeed(derived.keys[Roles.Dust]);
  const unshieldedKeystore = createKeystore(derived.keys[Roles.NightExternal], networkId);

  const wallet = await WalletFacade.init({
    configuration: {
      networkId,
      indexerClientConnection: { indexerHttpUrl: localDevnet.indexer, indexerWsUrl: localDevnet.indexerWS },
      provingServerUrl: new URL(localDevnet.proofServer),
      relayURL: new URL(localDevnet.node),
      txHistoryStorage: new InMemoryTransactionHistoryStorage(),
      costParameters: { additionalFeeOverhead: 300_000_000_000_000n, feeBlocksMargin: 5 },
    },
    shielded: async (c) => ShieldedWallet(c).startWithSecretKeys(shieldedSecretKeys),
    unshielded: async (c) =>
      UnshieldedWallet(c).startWithPublicKey(PublicKey.fromKeyStore(unshieldedKeystore)),
    dust: async (c) =>
      DustWallet(c).startWithSecretKey(dustSecretKey, ledger.LedgerParameters.initialParameters().dust),
  });
  await wallet.start(shieldedSecretKeys, dustSecretKey);
  const state = await wallet.waitForSyncedState();

  const zkConfigProvider = new NodeZkConfigProvider(zkConfigPath);
  const walletProvider = {
    getCoinPublicKey: () => state.shielded.coinPublicKey.toHexString(),
    getEncryptionPublicKey: () => state.shielded.encryptionPublicKey.toHexString(),
    async balanceTx() {
      throw new Error('e2e-check is read-only and should not balance transactions');
    },
    submitTx() {
      throw new Error('e2e-check is read-only and should not submit transactions');
    },
  } as any;

  const providers = {
    privateStateProvider: levelPrivateStateProvider({
      privateStateStoreName: 'hello-world-state',
      accountId: unshieldedKeystore.getBech32Address().toString(),
      privateStoragePasswordProvider: () => 'development',
    }),
    publicDataProvider: indexerPublicDataProvider(localDevnet.indexer, localDevnet.indexerWS),
    zkConfigProvider,
    proofProvider: httpClientProofProvider(localDevnet.proofServer, zkConfigProvider),
    walletProvider,
    midnightProvider: walletProvider,
  };

  // 3. Reconnect to the deployed contract and read its ledger state
  let found;
  try {
    found = await findDeployedContract(providers, {
      contractAddress: deployment.contractAddress,
      compiledContract: compiledContract as any,
    });
  } catch (err: any) {
    await wallet.stop();
    fail(`findDeployedContract threw: ${err?.message ?? err}`);
  }

  // The hello-world contract's ledger has a `message: Opaque<"string">`. Reading
  // the published state and confirming the field is *present* (the storeMessage
  // circuit hasn't run yet — the field exists but may be empty) is enough to
  // prove the contract is on chain and indexable.
  const ledgerState: any = (found as any).contractState ?? (found as any).publicState ?? null;
  if (!ledgerState) {
    await wallet.stop();
    fail('findDeployedContract returned no contract state.');
  }
  if (!Object.prototype.hasOwnProperty.call(ledgerState, 'message')) {
    await wallet.stop();
    fail(`contract state missing 'message' field; got keys: ${Object.keys(ledgerState).join(', ')}`);
  }

  console.log(`✅ e2e-check passed`);
  console.log(`   contractAddress: ${deployment.contractAddress}`);
  console.log(`   network:         ${deployment.network}`);
  console.log(`   ledger keys:     ${Object.keys(ledgerState).join(', ')}`);

  await wallet.stop();
  process.exit(0);
}

main().catch(async (err) => {
  console.error(err);
  process.exit(1);
});
```

Substitute `<LOCAL_NETWORK_ID>` and `<INDEXER_PATH>` with the literal Phase 0 values.

**Note on `findDeployedContract`'s return shape:** the property name holding the on-chain ledger view varies across SDK versions (`contractState`, `publicState`, etc.). The script tries both. If neither matches in the verification step below, inspect:

```bash
node -e "console.log(Object.keys(require('@midnight-ntwrk/midnight-js-contracts')))"
```

and adjust the `ledgerState` extraction accordingly.

- [ ] **Step 2: Add the test:e2e script**

In `templates/hello-world/package.json.template`, in the `"scripts"` block, add:

```json
"test:e2e": "tsx scripts/e2e-check.ts",
```

Place it between `"test"` and `"build"` for readability.

### Task 3.2: Manually verify the e2e check

- [ ] **Step 1: Build, scaffold, setup, run e2e**

```bash
npm run build
rm -rf /tmp/phase3-sandbox
node bin/create-midnight-app.js /tmp/phase3-sandbox -y -t hello-world
cd /tmp/phase3-sandbox
npm run setup
npm run test:e2e
```

Expected: `✅ e2e-check passed` line printed, exit 0.

If step fails on the `ledgerState` extraction, fix the script per the note in Task 3.1 Step 1, then re-run. Iterate until green.

- [ ] **Step 2: Tear down**

```bash
docker compose down -v
cd -
```

### Task 3.3: Commit Phase 3

- [ ] **Step 1: Stage and commit**

```bash
git add templates/hello-world/scripts/e2e-check.ts.template \
        templates/hello-world/package.json.template
git commit -m "$(cat <<'EOF'
feat(hello-world): add scripts/e2e-check.ts and npm run test:e2e

Smoke + read-back: reads deployment.json, reconnects via findDeployedContract,
reads the contract's ledger state. Exits non-zero with a structured error on
any failure. No test framework — runs directly via tsx so CI can call it as
a single command.

Refs: docs/superpowers/plans/2026-05-05-local-devnet-hello-world.md (Phase 3)

Co-Authored-By: Claude Code <noreply@anthropic.com>
EOF
)"
```

---

## Phase 4: Mirror config refactor into `cli.ts.template` and `check-balance.ts.template`

Goal: align the other two SDK-using scripts with the new network config so they keep working post-Phase-2.

### Task 4.1: Update `cli.ts.template`

**Files:**
- Modify: `templates/hello-world/src/cli.ts.template`

- [ ] **Step 1: Replace the CONFIG block and network-label default**

Find the `CONFIG = { ... }` object literal (~lines 36–42) and replace with the same `localDevnet` object as in `deploy.ts.template`:

```typescript
const localDevnet = {
  networkId: '<LOCAL_NETWORK_ID>' as const,
  indexer: 'http://127.0.0.1:8088<INDEXER_PATH>',
  indexerWS: 'ws://127.0.0.1:8088<INDEXER_PATH>/ws',
  node: 'ws://127.0.0.1:9944',
  proofServer: 'http://127.0.0.1:6300',
};
```

- [ ] **Step 2: Replace `setNetworkId('preprod')` with `setNetworkId(localDevnet.networkId)`**

- [ ] **Step 3: Replace `CONFIG.indexer`, `CONFIG.indexerWS`, `CONFIG.node`, `CONFIG.proofServer` references**

Search for `CONFIG.` usages in the file:

```bash
grep -n 'CONFIG\.' templates/hello-world/src/cli.ts.template
```

Replace each with the corresponding `localDevnet.` field.

- [ ] **Step 4: Update the printed network label default**

Find:
```typescript
console.log(`  Network: ${deployment.network || 'preprod'}\n`);
```
Replace with:
```typescript
console.log(`  Network: ${deployment.network || localDevnet.networkId}\n`);
```

- [ ] **Step 5: Update the wallet seed**

Replace any random-seed generation in `cli.ts.template` with the `GENESIS_SEED` constant (mirror Task 2.2). Add the same `LOCAL DEVNET ONLY` comment block above the constant.

### Task 4.2: Update `check-balance.ts.template`

**Files:**
- Modify: `templates/hello-world/src/check-balance.ts.template`

- [ ] **Step 1: Replace the CONFIG block, network ID, seed, and faucet line**

Apply the same three changes as Task 4.1 steps 1–3 and 5.

- [ ] **Step 2: Remove any "visit faucet at ..." line**

```bash
grep -n 'faucet' templates/hello-world/src/check-balance.ts.template
```

Delete every line that references the faucet URL or instructs the user to visit one.

### Task 4.3: Manually verify cli + check-balance still work end-to-end

- [ ] **Step 1: Build, scaffold, setup, run all three scripts**

```bash
npm run build
rm -rf /tmp/phase4-sandbox
node bin/create-midnight-app.js /tmp/phase4-sandbox -y -t hello-world
cd /tmp/phase4-sandbox
npm run setup
npm run test:e2e
npm run check-balance
# cli is interactive; just confirm it loads without error
echo "" | timeout 30 npx tsx src/cli.ts || true
```

Expected: `setup`, `test:e2e`, and `check-balance` all exit 0; `cli.ts` loads (showing the deployment info banner) before timing out from no input.

- [ ] **Step 2: Tear down**

```bash
docker compose down -v
cd -
```

### Task 4.4: Commit Phase 4

- [ ] **Step 1: Stage and commit**

```bash
git add templates/hello-world/src/cli.ts.template \
        templates/hello-world/src/check-balance.ts.template
git commit -m "$(cat <<'EOF'
refactor(hello-world): point cli.ts and check-balance.ts at local devnet

Same localDevnet config object and GENESIS_SEED constant as deploy.ts. Drops
the faucet URL from check-balance. The three scripts now agree on a single
source of truth for network configuration.

Refs: docs/superpowers/plans/2026-05-05-local-devnet-hello-world.md (Phase 4)

Co-Authored-By: Claude Code <noreply@anthropic.com>
EOF
)"
```

---

## Phase 5: Documentation + `_gitignore`

Goal: README and gitignore are accurate. A developer reading them gets the right mental model.

### Task 5.1: Update `templates/hello-world/_gitignore`

**Files:**
- Modify: `templates/hello-world/_gitignore`

- [ ] **Step 1: Read current contents**

```bash
cat templates/hello-world/_gitignore
```

- [ ] **Step 2: Remove the `.midnight-seed` line**

The deploy script no longer writes `.midnight-seed`. Drop the entry. Use Edit tool to delete the line; do not overwrite the rest of the file.

### Task 5.2: Rewrite `templates/hello-world/README.md.template`

**Files:**
- Modify: `templates/hello-world/README.md.template`

- [ ] **Step 1: Read current contents to preserve any tone/structure choices**

```bash
cat templates/hello-world/README.md.template
```

- [ ] **Step 2: Replace with the new README**

The new README:

```markdown
# {{projectName}}

A Midnight Network smart contract scaffolded with create-mn-app.

## Quick start

Requirements: Node 22, Docker (with Compose v2), and the Compact compiler at the version pinned in `.compact-version` at the create-mn-app repo root (the version this project was scaffolded against).

```bash
npm install
npm run setup
npm run test:e2e
```

`npm run setup` runs end-to-end with no prompts:

1. `docker compose up -d --wait` — starts a local Midnight devnet (node, indexer, proof-server) and blocks until all three pass their healthchecks.
2. `npm run compile` — compiles `contracts/hello-world.compact` to `contracts/managed/hello-world/`.
3. `npm run deploy` — derives the genesis-seed wallet (NIGHT pre-minted), registers UTXOs for DUST generation, deploys the contract, writes `deployment.json`.

`npm run test:e2e` reconnects to the deployed contract and reads its ledger state. Exits 0 if the contract is live and indexable.

## Local devnet

The project ships its own devnet via `docker-compose.yml`:

| Service        | Port | Purpose                                         |
| -------------- | ---- | ----------------------------------------------- |
| `node`         | 9944 | Midnight node, `dev` chain preset               |
| `indexer`      | 8088 | GraphQL indexer for chain state                 |
| `proof-server` | 6300 | Generates ZK proofs for contract transactions   |

State lives in container-managed volumes. Tear everything down with:

```bash
docker compose down -v
```

That removes all containers, networks, and volumes. The next `npm run setup` starts from a clean slate.

## ⚠️ LOCAL DEVNET ONLY

The deploy script uses a well-known genesis seed (`0000…0001`) so the
pre-minted NIGHT in the `dev` chain preset is immediately available. **Do
not use this seed against Preprod, mainnet, or any environment that
handles real value** — anyone running this devnet has full access to
funds at this seed.

## Available scripts

| Script               | Description                                                       |
| -------------------- | ----------------------------------------------------------------- |
| `npm run setup`      | One-shot: start devnet, compile, deploy.                          |
| `npm run compile`    | Compile the Compact contract.                                     |
| `npm run deploy`     | Deploy the compiled contract (requires devnet up + compiled).     |
| `npm run cli`        | Interactive CLI to call circuits on the deployed contract.        |
| `npm run check-balance` | Print the genesis-seed wallet's NIGHT and DUST balances.       |
| `npm run test:e2e`   | Smoke + read-back check against the deployed contract.            |
| `npm run clean`      | Remove `contracts/managed/` and `deployment.json`.                |
| `npm run proof-server:start` / `:stop` | Compose lifecycle for just the proof-server service. |

## Project structure

```
{{projectName}}/
├── contracts/
│   └── hello-world.compact     # Compact source
├── scripts/
│   └── e2e-check.ts            # smoke + read-back
├── src/
│   ├── cli.ts                  # interact with deployed contract
│   ├── deploy.ts               # local-devnet deploy
│   └── check-balance.ts        # NIGHT / DUST balance
├── docker-compose.yml          # node + indexer + proof-server
├── deployment.json             # written by deploy
├── package.json
└── tsconfig.json
```

## Compact compiler version

`.compact-version` at the create-mn-app repo root pinned the compiler
version this project was scaffolded against. To upgrade your local
compiler to that version:

```bash
compact self update
compact install <version>
compact use <version>
```
```

### Task 5.3: Update top-level `README.md`

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Update the "setup command does X" bullet list (around lines 18–24)**

Replace:
```markdown
The `setup` command:

1. Starts proof server via Docker
2. Compiles the Compact contract
3. Deploys to Preprod (prompts for faucet funding)

> Fund your wallet at [faucet.preprod.midnight.network](https://faucet.preprod.midnight.network/)
```

With:
```markdown
The `setup` command:

1. Starts a local Midnight devnet (node + indexer + proof-server) via Docker Compose, blocking until healthy
2. Compiles the Compact contract
3. Deploys to the local devnet using the dev preset's genesis-seed wallet (NIGHT pre-minted)

After setup, `npm run test:e2e` confirms the contract is on chain and indexable.
```

- [ ] **Step 2: Update the project-structure tree (around lines 116–129)**

Replace the existing tree with:
```markdown
```
my-app/
├── contracts/
│   └── hello-world.compact     # Compact smart contract
├── scripts/
│   └── e2e-check.ts            # smoke + read-back e2e test
├── src/
│   ├── cli.ts                  # Interact with deployed contract
│   ├── deploy.ts               # Deploy contract to local devnet
│   └── check-balance.ts        # Check wallet balance
├── docker-compose.yml          # Local devnet (node + indexer + proof-server)
├── package.json
└── deployment.json             # Generated after deploy (contract address)
```
```

- [ ] **Step 3: Audit other Preprod references**

```bash
grep -n -i 'preprod\|faucet' README.md
```

Update each remaining hit to reflect the local-devnet flow, EXCEPT for the requirements/options tables (which document the scaffolder, not the hello-world template specifically — non-hello-world templates may still target Preprod).

### Task 5.4: Commit Phase 5

- [ ] **Step 1: Stage and commit**

```bash
git add templates/hello-world/_gitignore \
        templates/hello-world/README.md.template \
        README.md
git commit -m "$(cat <<'EOF'
docs(hello-world): rewrite README for local-devnet flow; clean gitignore

Top-level README and the template's own README now describe the local
devnet setup, the genesis-seed warning, and `npm run test:e2e`. _gitignore
drops the `.midnight-seed` entry (no longer written).

Refs: docs/superpowers/plans/2026-05-05-local-devnet-hello-world.md (Phase 5)

Co-Authored-By: Claude Code <noreply@anthropic.com>
EOF
)"
```

---

## Phase 6: `.compact-version` + scaffolder tests

Goal: pin the compiler version, and update the scaffolder's own tests so they keep passing with the new file layout.

### Task 6.1: Add `.compact-version`

**Files:**
- Create: `.compact-version`

- [ ] **Step 1: Write the file**

Use `COMPACT_VERSION` from Phase 0.1. The file is exactly one line, the version number, no trailing newline tooling-special characters:

```bash
printf '%s\n' "$COMPACT_VERSION" > .compact-version
```

(Substitute the literal version, e.g. `0.31.0`.)

- [ ] **Step 2: Verify**

```bash
cat .compact-version
```

Expected: a single version line.

### Task 6.2: Update `template-manager.test.ts` (TDD)

**Files:**
- Modify: `src/__tests__/template-manager.test.ts`

- [ ] **Step 1: Read the current test for hello-world**

```bash
sed -n '1,80p' src/__tests__/template-manager.test.ts
```

Locate the `it(...)` block that asserts hello-world's expected files exist after scaffold (per the explore agent's report, around lines 23–49).

- [ ] **Step 2: Write a failing test for the new file**

In the same `it` block (or a new one if cleanly separable), add an assertion:

```typescript
expect(fs.existsSync(path.join(projectPath, 'scripts', 'e2e-check.ts'))).toBe(true);
```

Place it next to the existing file-presence checks.

- [ ] **Step 3: Run the test, verify it passes**

```bash
npm run test -- template-manager
```

Expected: PASS — `scripts/e2e-check.ts.template` was added to the template in Phase 3, so the rendered project does include `scripts/e2e-check.ts`. (This is technically not failing-then-passing TDD since the implementation already exists from Phase 3; the value of this test is permanent regression coverage.)

If the test FAILS — that means the scaffolder didn't render the file — debug. The `template-manager.ts` walks recursively, so a `scripts/` subdirectory should "just work."

### Task 6.3: Update `src/test.ts`

**Files:**
- Modify: `src/test.ts`

- [ ] **Step 1: Locate the `requiredFiles` list**

```bash
grep -n 'requiredFiles\|required_files' src/test.ts
```

- [ ] **Step 2: Add `scripts/e2e-check.ts` to the list**

Use the Edit tool. The exact insertion location is the same array that already lists files like `src/deploy.ts`, `package.json`, etc.

- [ ] **Step 3: Run the smoke test**

```bash
npm run test:smoke
```

Expected: PASS.

### Task 6.4: Commit Phase 6

- [ ] **Step 1: Stage and commit**

```bash
git add .compact-version src/__tests__/template-manager.test.ts src/test.ts
git commit -m "$(cat <<'EOF'
chore: pin Compact compiler in .compact-version; cover e2e-check in tests

The new .compact-version file is the single source of truth for which
Compact compiler this scaffolder is tested against — the e2e CI workflows
read it. template-manager.test.ts and the integration smoke test now
require scripts/e2e-check.ts to be rendered into the scaffolded project.

Refs: docs/superpowers/plans/2026-05-05-local-devnet-hello-world.md (Phase 6)

Co-Authored-By: Claude Code <noreply@anthropic.com>
EOF
)"
```

---

## Phase 7: CI workflows

Goal: three trigger workflows + one shared reusable workflow that runs the full pipeline on `ubuntu-latest` and `macos-latest`.

### Task 7.1: Create the reusable workflow `_e2e-job.yml`

**Files:**
- Create: `.github/workflows/_e2e-job.yml`

- [ ] **Step 1: Write the file**

The reusable workflow takes two inputs: `runs-on` and `scaffold-command`. The boolean "build the scaffolder from source" defaults true; the post-publish flavor sets it false.

```yaml
name: E2E (reusable)

on:
  workflow_call:
    inputs:
      runs-on:
        description: Runner to use (ubuntu-latest | macos-latest)
        required: true
        type: string
      scaffold-command:
        description: Command that scaffolds a fresh project at ./test-app
        required: true
        type: string
      build-source:
        description: Whether to npm ci + npm run build the scaffolder source
        required: false
        type: boolean
        default: true

jobs:
  e2e:
    runs-on: ${{ inputs.runs-on }}
    timeout-minutes: 45

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Node 22
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Set up Docker (macOS only)
        if: runner.os == 'macOS'
        run: |
          # MACOS_DOCKER_SETUP from Phase 0.6 — substitute the chosen approach.
          # Option B (brew + colima) shown here:
          brew install docker docker-compose colima
          colima start --cpu 2 --memory 4 --disk 30
          docker version

      - name: Read pinned Compact compiler version
        id: compact-version
        run: echo "version=$(cat .compact-version)" >> "$GITHUB_OUTPUT"

      - name: Install Compact compiler
        run: |
          # Use the same official installer as a developer machine; pin the version.
          curl -fsSL https://compact.midnight.network/install.sh | bash -s -- --no-prompt
          # Make sure the compact CLI is on PATH for subsequent steps.
          echo "$HOME/.compact/bin" >> "$GITHUB_PATH"
          export PATH="$HOME/.compact/bin:$PATH"
          compact self update --no-prompt || true
          compact install ${{ steps.compact-version.outputs.version }}
          compact use ${{ steps.compact-version.outputs.version }}
          compact --version

      - name: Build scaffolder (source flavor)
        if: ${{ inputs.build-source }}
        run: |
          npm ci
          npm run build

      - name: Scaffold test-app
        run: |
          rm -rf test-app
          ${{ inputs.scaffold-command }}
          test -d test-app

      - name: npm run setup
        working-directory: test-app
        run: npm run setup

      - name: npm run test:e2e
        working-directory: test-app
        run: npm run test:e2e

      - name: Assert deployment.json
        working-directory: test-app
        run: |
          jq -e '.contractAddress | type == "string" and length > 0' deployment.json
          jq -r '.contractAddress, .network' deployment.json

      - name: Capture diagnostics on failure
        if: failure()
        working-directory: test-app
        run: |
          docker compose ps -a > docker-ps.txt 2>&1 || true
          docker compose logs --no-color > docker-logs.txt 2>&1 || true

      - name: Upload diagnostics artifact on failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: e2e-diagnostics-${{ inputs.runs-on }}
          path: |
            test-app/docker-ps.txt
            test-app/docker-logs.txt
            test-app/deployment.json
            test-app/contracts/managed/
          if-no-files-found: ignore

      - name: Tear down devnet
        if: always()
        working-directory: test-app
        run: docker compose down -v || true
```

**If `MACOS_DOCKER_SETUP` from Phase 0.6 is Option A (third-party action), replace the macOS step with:**

```yaml
      - name: Set up Docker (macOS only)
        if: runner.os == 'macOS'
        uses: <chosen-action>@<chosen-tag>
```

with the action and tag identified in Phase 0.6.

**If the Compact installer URL or installer arguments differ from `https://compact.midnight.network/install.sh --no-prompt`, fix the "Install Compact compiler" step to match the current official instructions.** Verify by running:

```bash
curl -fsSL https://compact.midnight.network/install.sh | head -50
```

before committing.

### Task 7.2: Create `e2e.yml`

**Files:**
- Create: `.github/workflows/e2e.yml`

- [ ] **Step 1: Write the file**

```yaml
name: E2E

on:
  pull_request:
    branches: [main]
    paths:
      - 'templates/**'
      - 'src/**'
      - 'bin/**'
      - 'package.json'
      - 'package-lock.json'
      - '.compact-version'
      - '.github/workflows/_e2e-job.yml'
      - '.github/workflows/e2e.yml'
  push:
    branches: [main]
    paths:
      - 'templates/**'
      - 'src/**'
      - 'bin/**'
      - 'package.json'
      - 'package-lock.json'
      - '.compact-version'
      - '.github/workflows/_e2e-job.yml'
      - '.github/workflows/e2e.yml'
  workflow_dispatch:

jobs:
  matrix:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest]
    uses: ./.github/workflows/_e2e-job.yml
    with:
      runs-on: ${{ matrix.os }}
      scaffold-command: 'node bin/create-midnight-app.js test-app -y -t hello-world'
      build-source: true
```

### Task 7.3: Create `e2e-nightly.yml`

**Files:**
- Create: `.github/workflows/e2e-nightly.yml`

- [ ] **Step 1: Write the file**

```yaml
name: E2E Nightly

on:
  schedule:
    - cron: '0 3 * * *'
  workflow_dispatch:

jobs:
  matrix:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest]
    uses: ./.github/workflows/_e2e-job.yml
    with:
      runs-on: ${{ matrix.os }}
      scaffold-command: 'node bin/create-midnight-app.js test-app -y -t hello-world'
      build-source: true
```

### Task 7.4: Create `e2e-postpublish.yml`

**Files:**
- Create: `.github/workflows/e2e-postpublish.yml`

- [ ] **Step 1: Write the file**

```yaml
name: E2E Post-Publish

on:
  workflow_run:
    workflows: ['Publish to NPM']
    types: [completed]

jobs:
  matrix:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest]
    uses: ./.github/workflows/_e2e-job.yml
    with:
      runs-on: ${{ matrix.os }}
      scaffold-command: 'npx --yes create-mn-app@latest test-app -y -t hello-world'
      build-source: false
```

### Task 7.5: Validate the workflows locally before pushing

- [ ] **Step 1: Lint the workflow files**

```bash
# If actionlint is installed:
actionlint .github/workflows/e2e.yml \
           .github/workflows/e2e-nightly.yml \
           .github/workflows/e2e-postpublish.yml \
           .github/workflows/_e2e-job.yml
# If not installed, brew install actionlint or skip and rely on the first PR.
```

Expected: no errors.

- [ ] **Step 2: Verify the reusable-workflow path resolves**

```bash
test -f .github/workflows/_e2e-job.yml
grep -n 'uses: ./.github/workflows/_e2e-job.yml' .github/workflows/e2e.yml \
                                                 .github/workflows/e2e-nightly.yml \
                                                 .github/workflows/e2e-postpublish.yml
```

Expected: each trigger workflow references the reusable workflow at the same relative path.

### Task 7.6: Commit Phase 7

- [ ] **Step 1: Stage and commit**

```bash
git add .github/workflows/_e2e-job.yml \
        .github/workflows/e2e.yml \
        .github/workflows/e2e-nightly.yml \
        .github/workflows/e2e-postpublish.yml
git commit -m "$(cat <<'EOF'
ci: add end-to-end workflows (PR/push, nightly, post-publish)

Three trigger workflows call a shared reusable _e2e-job.yml that scaffolds
hello-world, runs `npm run setup`, runs `npm run test:e2e`, and asserts
deployment.json. Matrix is [ubuntu-latest, macos-latest]; macOS provisions
Docker via Colima before the devnet starts.

- e2e.yml: PR + push to main + manual dispatch, path-filtered
- e2e-nightly.yml: 03:00 UTC cron, catches drift in pinned external resources
- e2e-postpublish.yml: workflow_run on "Publish to NPM" success, scaffolds
  via `npx create-mn-app@latest` to verify the published tarball

Refs: docs/superpowers/plans/2026-05-05-local-devnet-hello-world.md (Phase 7)

Co-Authored-By: Claude Code <noreply@anthropic.com>
EOF
)"
```

### Task 7.7: Push the branch and watch CI

- [ ] **Step 1: Push**

```bash
git push -u origin <branch-name>
```

(Use whichever branch was created at the start of execution.)

- [ ] **Step 2: Watch the run**

```bash
gh run watch
```

or

```bash
gh run list --workflow=e2e.yml --limit 1
gh run view <run-id> --log
```

Expected: both `ubuntu-latest` and `macos-latest` jobs end in success. The PR's status checks include `E2E / matrix (ubuntu-latest)` and `E2E / matrix (macos-latest)`.

If a run fails, download the diagnostics artifact:

```bash
gh run download <run-id> -n e2e-diagnostics-ubuntu-latest
gh run download <run-id> -n e2e-diagnostics-macos-latest
```

and use `docker-logs.txt`, `deployment.json` to triage.

### Task 7.8: Open the PR

- [ ] **Step 1: Create PR**

```bash
gh pr create --title "feat(hello-world): bundle local devnet, verify end-to-end in CI" \
  --body "$(cat <<'EOF'
## Summary

- Replaces the Preprod-targeted hello-world template with a fully local, bundled devnet (node + indexer + proof-server)
- `npm run setup` now runs end-to-end with no prompts and produces a deployed contract every time
- New `npm run test:e2e` reconnects to the deployed contract and reads its ledger state
- Three CI workflows verify the entire pipeline on Linux and macOS — on every PR/push, nightly, and after every npm publish

## Test plan

- [x] Local: `npm run build && node bin/create-midnight-app.js sandbox -y -t hello-world && cd sandbox && npm run setup && npm run test:e2e` — green
- [x] Local: `npm run test` (vitest) and `npm run test:smoke` — green
- [ ] CI: `e2e.yml` matrix (ubuntu-latest, macos-latest) — green on this PR
- [ ] First scheduled run of `e2e-nightly.yml` (post-merge) — green
- [ ] First release after merge: `e2e-postpublish.yml` — green

Spec: `docs/superpowers/plans/2026-05-05-local-devnet-hello-world.md`

Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

---

## Self-review checklist (run before declaring the plan done)

1. **Spec coverage** — every D1–D10 in the spec is implemented in a numbered task above?
   - D1 (bundle compose): Phase 2 / Task 2.1 ✓
   - D2 (pin everything): Phase 0 / Tasks 0.1–0.3 + Phase 6 / Task 6.1 ✓
   - D3 (full replacement, design extensibly): Phase 1 (extract config) + Phase 2 (flip) — the `localDevnet`-shaped object is the seam ✓
   - D4 (3-service stack): Task 2.1 ✓
   - D5 (genesis seed, ephemeral): Task 2.2 ✓
   - D6 (smoke + read-back): Phase 3 ✓
   - D7 (non-interactive): Phase 1 (prompts removed) ✓
   - D8 (3 workflows + reusable): Phase 7 ✓
   - D9 (matrix + Node 22): Tasks 7.1, 7.2, 7.3, 7.4 ✓
   - D10 (`.compact-version`): Task 6.1 ✓
2. **Placeholder scan** — `<NODE_IMAGE_TAG>`, `<INDEXER_IMAGE_TAG>`, `<PROOF_SERVER_IMAGE_TAG>`, `<LOCAL_NETWORK_ID>`, `<INDEXER_PATH>`, `<chosen-action>` are all explicit references to Phase 0 outputs, not "TBD"s. Each task that uses them tells the engineer where the value came from.
3. **Type/name consistency** — `localDevnet` (object), `GENESIS_SEED` (constant), `findDeployedContract`, `deployment.json` keys (`contractAddress`, `network`, `deployedAt`, `status`) used identically across deploy.ts, cli.ts, check-balance.ts, e2e-check.ts.
4. **Out-of-scope items not snuck back in** — no fresh-random ephemeral wallet (D5), no multi-network selector (D3), no vitest e2e harness (D6), no Windows runner (D9), no `CI=true` branching (D7), no faucet service in compose, no persistent volumes.

---

## Notes for the executor

- **Phase 0 is mandatory before any other phase.** Don't guess at versions, image tags, or network IDs. The system has flagged training data on Midnight as unreliable.
- **Each phase ends with a commit.** This is the rollback granularity — if Phase 5 breaks something, `git revert HEAD` returns to a known-green state without losing Phases 1–4.
- **Manual end-to-end runs are the verification mechanism for template content changes.** There's no unit-test substitute for "spin up the devnet and watch a contract get deployed." Phases 2, 3, 4 each end with this dance.
- **The PR introducing Phase 7 is the first run of `e2e.yml`.** That run on this branch is the integration test for the entire change — both runners must be green before merge.
