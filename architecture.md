# Sentinel Architecture

## Overview

Sentinel currently has two architectural layers in the same repository:

- The root JavaScript MVP, which is the executable path validated by tests.
- The `og-integration/` TypeScript package, which is mostly a scaffold for future 0G-native integrations.

If you are trying to understand what works today, start with the root JavaScript layer.

## Runtime Components

### Contracts

- [contracts/InferenceGuard.sol](/home/laddu/Sentinel/contracts/InferenceGuard.sol)
  Stores `Proof` objects keyed by `executionId`.
  Public flow:
  `submitProof(bytes32 executionId, bytes32 rootHash)`
  `isProofValid(bytes32 executionId)`
  `consumeProof(bytes32 executionId)`

- [contracts/SentinelINFT.sol](/home/laddu/Sentinel/contracts/SentinelINFT.sol)
  ERC-721 token that tracks:
  `experienceCycles`
  `storagePointer`
  `strategyFingerprint`
  `active`

- [contracts/PositionRegistry.sol](/home/laddu/Sentinel/contracts/PositionRegistry.sol)
  Registers:
  `registerLendingPosition(address, Protocol, uint256)`
  `registerUniswapPosition(address, int24, int24)`

- [contracts/MockUniswapV3Pool.sol](/home/laddu/Sentinel/contracts/MockUniswapV3Pool.sol)
  Demo pool with a Uniswap V3-like `slot0()` interface and owner-controlled tick changes.

### Agents and Services

- [yieldAgent.js](/home/laddu/Sentinel/yieldAgent.js)
  Polls LP positions from `PositionRegistry`.
  Reads `slot0()` from each pool.
  Emits proposals through `yieldEmitter`.

- [agents/coordinator/index.js](/home/laddu/Sentinel/agents/coordinator/index.js)
  Attaches to `yieldEmitter`.
  Classifies and queues proposals.
  Ensures proof presence in `InferenceGuard`.
  Creates and triggers KeeperHub workflows.
  Consumes proof and increments iNFT experience.

- [agents/coordinator/proposalQueue.js](/home/laddu/Sentinel/agents/coordinator/proposalQueue.js)
  In-memory queue with `push`, `pop`, `peek`, `size`, and `clear`.
  Sorts by numeric priority, then timestamp.

- [agents/coordinator/priorityEngine.js](/home/laddu/Sentinel/agents/coordinator/priorityEngine.js)
  Maps agent type to priority constants:
  `RISK = 1`
  `YIELD = 2`
  `HOLD = 3`

- [keeperhub/client.js](/home/laddu/Sentinel/keeperhub/client.js)
  Chooses MCP first when `mcpEnabled === true` and MCP health check passes.
  Falls back to REST otherwise.

- [keeperhub/restClient.js](/home/laddu/Sentinel/keeperhub/restClient.js)
  Sends:
  `POST /v1/workflows`
  `POST /v1/workflows/:id/execute`
  `POST /v1/marketplace/publish`

- [keeperhub/mcpClient.js](/home/laddu/Sentinel/keeperhub/mcpClient.js)
  Calls local MCP endpoints:
  `/health`
  `/mcp/keeperhub/create_workflow`
  `/mcp/keeperhub/trigger_execution`

- [workflows/workflowPublisher.js](/home/laddu/Sentinel/workflows/workflowPublisher.js)
  Reads workflow JSON definitions and publishes them through KeeperHub.

## Proposal Shape

The proposal emitted by `yieldAgent.js` looks like this:

```js
{
  agentType: "YIELD",
  action: "REBALANCE_LP",
  priority: "HIGH" | "MEDIUM",
  positionId,
  poolAddress,
  poolLabel,
  currentTick,
  currentRange: { tickLower, tickUpper },
  suggestedRange: { tickLower, tickUpper },
  distanceFromRange,
  reasoning,
  timestamp,
  executionId
}
```

The Coordinator later enriches it with:

```js
{
  ...proposal,
  priority: 2,
  enrichedAt: <timestamp>,
  status: "PENDING"
}
```

## Implemented Request Flow

### 1. Position registration

A user registers a lending or LP monitoring position in `PositionRegistry`.

### 2. Yield detection

`yieldAgent.js`:

- reads the monitored user's positions
- filters to active `UNISWAP_V3` entries
- reads `slot0().tick` from each pool
- emits a proposal if the tick is outside the configured range

### 3. Queueing and prioritization

The Coordinator listens for emitted proposals and:

- classifies them with `priorityEngine.js`
- pushes them into `ProposalQueue`
- processes the highest-priority item first

### 4. Proof preparation

The Coordinator checks whether a proof already exists on `InferenceGuard`.
If not, it builds a local `rootHash` from proposal data and submits it on-chain.

This is an important implementation detail:
the current root JavaScript MVP does not fetch a proof from a real 0G DA pipeline.
It creates the proof hash locally from the proposal payload.

### 5. KeeperHub execution

If `isProofValid(executionId)` returns true, the Coordinator:

- creates a workflow on KeeperHub
- triggers execution of that workflow

If KeeperHub creation fails, the Coordinator still assigns a fallback workflow id.
If triggering fails, it records an `api_unavailable` result and continues cleanup.

### 6. Post-execution cleanup

After the KeeperHub attempt, the Coordinator:

- calls `consumeProof(executionId)` on `InferenceGuard`
- calls `incrementExperience(tokenId)` on `SentinelINFT`

## Workflow Definitions

Two workflow definitions live under [workflows](/home/laddu/Sentinel/workflows):

- [sparkLiquidationShield.json](/home/laddu/Sentinel/workflows/sparkLiquidationShield.json)
- [lpRangeRebalancer.json](/home/laddu/Sentinel/workflows/lpRangeRebalancer.json)

These are packaged and published by `workflowPublisher.js`.

Important naming note:
the file in this repository is `lpRangeRebalancer.json`.
Any reference to `lpRebalancer.json` is out of date.

## Testing Architecture

The repo now has three testing layers:

### Contract tests

[test/contracts/sentinel.test.js](/home/laddu/Sentinel/test/contracts/sentinel.test.js)

Uses Hardhat's in-process network.
Deploys fresh contracts per suite.
No external RPC or live network calls.

### Unit tests

[test/unit](/home/laddu/Sentinel/test/unit)

Stubs `ethers`, emitters, and logger behavior to validate:

- Yield Agent detection logic
- Coordinator sequencing
- Queue ordering
- Priority classification

### Mocked integration and E2E tests

[test/integration](/home/laddu/Sentinel/test/integration)

Mocks KeeperHub calls and runs an end-to-end local simulation:

1. deploy contracts on Hardhat
2. register an LP position
3. move the mock pool out of range
4. emit a Yield Agent proposal
5. process it through the Coordinator
6. consume the proof
7. increment experience

## What Is Not Fully Implemented Yet

The `og-integration/` package suggests a broader architecture than the MVP currently executes.

Today, these areas are still scaffolded or mocked there:

- 0G Compute calls
- 0G Storage DAG persistence
- 0G DA writes and proofs
- cryptographic TEE verification
- fallback model integrations

That package is useful as a design direction, but it should not be treated as the currently validated runtime path.

## Practical Entry Points

If you want to work on the currently tested system, start here:

- [runSystem.js](/home/laddu/Sentinel/runSystem.js)
- [yieldAgent.js](/home/laddu/Sentinel/yieldAgent.js)
- [agents/coordinator/index.js](/home/laddu/Sentinel/agents/coordinator/index.js)
- [contracts](/home/laddu/Sentinel/contracts)
- [test](/home/laddu/Sentinel/test)

If you want to continue the future 0G-native buildout, inspect:

- [og-integration/src/index.ts](/home/laddu/Sentinel/og-integration/src/index.ts)
- [og-integration/src/services](/home/laddu/Sentinel/og-integration/src/services)
- [og-integration/src/pipeline/executor.ts](/home/laddu/Sentinel/og-integration/src/pipeline/executor.ts)
