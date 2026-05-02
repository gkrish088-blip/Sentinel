# Sentinel

Sentinel is a DeFi position guardian built around a small on-chain proof gate, an iNFT identity contract, a position registry, a rule-based Yield Agent, and a Coordinator that routes approved actions to KeeperHub.

The current repo contains two layers:

- A working JavaScript MVP in the repo root.
- A larger `og-integration/` TypeScript scaffold with mock 0G services and future-facing code.

This README describes the implementation that is actually present and now covered by tests.

## What Is Implemented

- `InferenceGuard.sol` stores a `rootHash` by `executionId`, reports whether a proof exists and is unconsumed, and marks proofs as consumed.
- `SentinelINFT.sol` mints an ERC-721 iNFT-style token with `storagePointer`, `strategyFingerprint`, and `experienceCycles`.
- `PositionRegistry.sol` stores lending positions and Uniswap V3-style LP monitoring ranges.
- `MockUniswapV3Pool.sol` exposes `slot0()` and owner-controlled tick movement for demos.
- `yieldAgent.js` watches registered Uniswap V3-style positions and emits a YIELD rebalance proposal when the pool tick leaves the registered range.
- `agents/coordinator/index.js` receives proposals, ensures a proof exists in `InferenceGuard`, calls KeeperHub, consumes the proof, and increments iNFT experience.
- `workflows/workflowPublisher.js` creates and publishes the Spark and LP workflow JSON definitions to KeeperHub.

## Verified Behavior

The repo now includes local tests under `test/` and they pass with:

```bash
npx hardhat test
```

Current result:

- `48 passing`

Verified by tests:

- `InferenceGuard` stores proofs, emits `ProofSubmitted`, and marks proofs consumed.
- `SentinelINFT` mints correctly and increments `experienceCycles`.
- `PositionRegistry` stores both lending and LP monitoring records.
- `MockUniswapV3Pool` returns the expected `slot0()` shape and restricts tick changes to the owner.
- `yieldAgent.js` emits proposals only when a pool is out of range.
- `agents/coordinator/index.js` submits proofs, validates them, calls KeeperHub, consumes proofs, and increments iNFT experience.
- `keeperhub/client.js` routes between MCP and REST based on `mcpEnabled`.
- `workflows/workflowPublisher.js` reads workflow JSON and sends the expected payloads.
- A mocked end-to-end simulation completes the full Yield Agent -> Coordinator -> KeeperHub -> proof consume -> experience increment cycle.

## Current Flow

The implemented MVP path is:

1. A Uniswap V3-style LP position is registered in `PositionRegistry`.
2. `yieldAgent.js` polls the pool's `slot0()` tick.
3. If the tick is outside `[tickLower, tickUpper]`, the agent emits a proposal with:
   `agentType`, `action`, `positionId`, `poolAddress`, `currentTick`, `currentRange`, `suggestedRange`, `reasoning`, `timestamp`, and `executionId`.
4. The Coordinator classifies the proposal, queues it, and decides whether to execute it.
5. The Coordinator calls `InferenceGuard.submitProof(executionId, rootHash)`.
6. If `InferenceGuard.isProofValid(executionId)` is true, the Coordinator calls KeeperHub to create and trigger a workflow.
7. After execution, the Coordinator calls `InferenceGuard.consumeProof(executionId)`.
8. The Coordinator calls `SentinelINFT.incrementExperience(tokenId)`.

## Important Limits

Some repo claims are still ahead of the implemented code:

- The root JavaScript MVP does not perform real 0G Compute, 0G Storage, or 0G DA integration.
- The `og-integration/` package contains mocks, placeholders, and TODOs for those services.
- The proof submitted by the root Coordinator is currently derived locally from proposal data, not from a real DA write.
- `keeperhub/client.js` switches by constructor config `mcpEnabled`; it does not read a `USE_MCP` environment variable.
- The workflow file in this repo is `workflows/lpRangeRebalancer.json`, not `lpRebalancer.json`.
- `yieldAgent.js` lives at the repo root, not under `agents/`.

## Repo Layout

```text
Sentinel/
  contracts/
    InferenceGuard.sol
    SentinelINFT.sol
    PositionRegistry.sol
    MockUniswapV3Pool.sol
  agents/coordinator/
    index.js
    proposalQueue.js
    priorityEngine.js
  keeperhub/
    client.js
    restClient.js
    mcpClient.js
    retryHandler.js
    rpcFallover.js
    x402Payment.js
  workflows/
    sparkLiquidationShield.json
    lpRangeRebalancer.json
    workflowPublisher.js
  scripts/
    deploy.js
    deployMockPool.js
    demoTrigger.js
    demoReset.js
  yieldAgent.js
  runSystem.js
  og-integration/
  test/
```

## Local Usage

Install dependencies:

```bash
npm install
```

Compile contracts:

```bash
npx hardhat compile
```

Run tests:

```bash
npx hardhat test
```

Deploy the demo contracts to 0G Galileo:

```bash
npx hardhat run scripts/deploy.js --network galileo
npx hardhat run scripts/deployMockPool.js --network galileo
```

Run the integrated demo loop locally against deployed addresses:

```bash
node runSystem.js
```

Move the mock pool out of range:

```bash
npx hardhat run scripts/demoTrigger.js --network galileo
```

Move the mock pool back in range:

```bash
npx hardhat run scripts/demoTrigger.js --network galileo -- --reset
```

## More Detail

See [architecture.md](/home/laddu/Sentinel/architecture.md) for a code-level architecture map, request flow, and the difference between the implemented MVP and the future `og-integration/` layer.
