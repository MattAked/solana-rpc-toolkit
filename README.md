# Solana RPC Toolkit

An asynchronous Python toolkit for retrieving and streaming Solana blockchain data through **JSON-RPC** and **Yellowstone gRPC**. Built as part of a broader on-chain data research workflow, with an emphasis on handling RPC requests, consuming live account/transaction updates, and structuring raw responses for downstream analysis.

> **Project scope:** This is a portfolio release of a personal engineering project, not a maintained production SDK. It requires access to a compatible Solana RPC provider; some features depend on provider support and may require adaptation.

## What it does

- **Asynchronous JSON-RPC access:** `RPCClient` wraps common Solana RPC methods and manages asynchronous requests using `aiohttp`.
- **Multiple RPC endpoints:** Configure endpoints with their requests-per-second limits; the request manager distributes batches across available endpoints and includes retry/error handling.
- **Historical data retrieval:** Helpers for address transaction signatures, pagination, and signature/slot-based historical queries.
- **Yellowstone gRPC streaming:** Subscribe to transactions involving selected accounts or to token-account changes, using generated protobuf bindings.
- **Structured blockchain data:** Response models and optional DEX-specific parsing utilities to support wallet-level analysis and trade-data workflows.

The core focus is **data access and streaming infrastructure**. Transaction interpretation is included as a supporting utility, not a guarantee of complete or unambiguous trade classification. Complex Solana transactions, multi-program interactions and token balance changes can make wallet intent difficult to infer; parsers may miss or misclassify activity.

## Repository layout

```text
argus_rpc/
├── RPClient.py                # Higher-level JSON-RPC retrieval helpers
├── RPCRequestManager.py       # Multi-endpoint request distribution
├── AsyncRPCEndpoint.py        # Async HTTP sessions, pacing and retries
├── gRPCClient.py              # Yellowstone gRPC connection/subscription base
├── AccountsTxStream.py       # Account-filtered transaction updates
├── AccountsChangesStream.py  # Token-account change subscriptions
├── generated/                # Generated protobuf client bindings
└── utils/                    # RPC request/response models and parsing helpers
```

## Installation

Requires Python and a compatible Solana JSON-RPC endpoint. For streaming, a Yellowstone gRPC endpoint and its access token are also required.

```bash
python -m venv .venv
# Windows: .venv\Scripts\activate
# macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt
pip install -e .
```

## JSON-RPC example

Supply your endpoint through an environment variable rather than committing a private provider URL or API key.

```python
import asyncio
import os

from argus_rpc.RPClient import RPCClient

async def main():
    endpoint = os.environ["SOLANA_RPC_URL"]
    client = RPCClient(endpoints_list=[(endpoint, 10)])  # endpoint, request limit/sec
    try:
        signatures = await client.get_tx_signatures(
            "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA",
            limit=10,
        )
        for signature in signatures or []:
            print(signature.signature)
    finally:
        await client.close()

asyncio.run(main())
```

The `RPCClient` also exposes request-manager functionality for sending individual requests or distributing batches across configured endpoints. Requests and response types are defined under `argus_rpc/utils/RPC/`.

## Streaming example

`AccountsTxStream` accepts a mapping of filter names to **lists** of account addresses. The example below monitors transactions involving the SPL Token Program account; choose filters appropriate to your workload.

```python
import asyncio
import os

from argus_rpc.AccountsTxStream import AccountsTxStream

async def main():
    stream = AccountsTxStream(
        endpoint=os.environ["SOLANA_GRPC_ENDPOINT"],
        token=os.environ["SOLANA_GRPC_TOKEN"],
        accounts={"token_program": ["TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA"]},
    )
    try:
        async for update in stream.start_monitoring():
            print(update.transaction.slot)
    finally:
        stream.close()

asyncio.run(main())
```

**Note:** Yellowstone gRPC subscription semantics, endpoint authentication, throughput and replay support depend on the provider. The existing gRPC implementation uses a synchronous streaming stub inside an async generator, so it may block the event loop under some conditions; robust production deployments should revisit this design and implement supervised reconnection.

## Design notes and limitations

- Endpoint request pacing, retry logic and batch distribution are implemented in `AsyncRPCEndpoint` and `RPCRequestManager`; they are not a guarantee of strict global rate limiting or lossless retrieval.
- Stream subscription classes provide basic connection monitoring and filtering; automatic end-to-end gap recovery is **not** implemented by this package.
- Some parsing utilities target specific Solana DEX programs. They are best treated as exploratory extraction helpers and validated against raw transactions before analytical or trading use.
- This repository does not include automated test coverage or benchmark results. Provider APIs and protobuf compatibility can change over time.
- RPC endpoint URLs can contain credentials. Avoid sharing logs or exceptions containing full provider URLs, and keep endpoints and tokens outside source control.

## Background

Originally developed to support Solana on-chain research and the construction of wallet-oriented trading datasets. This public version focuses on the reusable RPC access, streaming and data-processing components, with development-only scripts excluded.
