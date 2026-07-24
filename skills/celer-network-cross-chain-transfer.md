---
name: Estimate and track a cBridge cross-chain transfer
description: Use the cBridge gateway API to discover supported chains/tokens, estimate a cross-chain transfer, and track its status to completion or refund.
api: grpc/celer-network-gateway.proto
operations: [GetTransferConfigs, GetTokenInfo, EstimateAmt, GetTransferStatus, TransferHistory]
---

# Estimate and track a cBridge cross-chain transfer

The cBridge gateway (`sgn.gateway.v1.Web`) is a public gRPC-Web/REST service. Read
operations need no credential. Base URL: `https://cbridge-prod2.celer.app`
(mainnet) or `https://cbridge-v2-test.celer.network` (testnet).

## Steps

1. **Discover chains and tokens** — call `GetTransferConfigs`
   (`GET /v1/getTransferConfigs`). Record each `Chain.id` and every token's
   `decimal` — you need the decimals to format `amt` correctly.
2. **(Optional) Inspect one token** — call `GetTokenInfo`
   (`GET /v1/getTokenInfo`) with `chain_id` + `token_symbol`.
3. **Estimate the transfer** — call `EstimateAmt` (`GET /v1/estimateAmt`) with
   `src_chain_id`, `dst_chain_id`, `token_symbol`, `amt`, `usr_addr`, and
   `slippage_tolerance` (slippage × 1M, e.g. 0.5% = 5000). Read back
   `eq_value_token_amt`, `perc_fee`, and `base_fee`; compute
   `minimum_received_amt = eq_value_token_amt × (1 − slippage_tolerance) − fee`.
4. **Send the on-chain transfer** — the actual token movement is an on-chain
   contract call from the user's wallet to the bridge `contract_addr` from step 1
   (out of scope for this gateway API).
5. **Track status** — call `GetTransferStatus`
   (`POST /v1/getTransferStatus`, body `{ transfer_id }`). Poll until
   `status` reaches `TRANSFER_COMPLETED`, or `TRANSFER_TO_BE_REFUNDED` (then
   read `refund_reason`, e.g. `BAD_LIQUIDITY` / `BAD_SLIPPAGE` / `BAD_TOKEN`).
6. **Review history** — call `TransferHistory` (`GET /v1/transferHistory`) with
   `addr`; page with `next_page_token` (empty string for the first page) and
   `page_size`.

## Rules

- **Error envelope**: every response carries `err { code, msg }`. `code` 0
  (`ERROR_CODE_UNDEFINED`) is success; `1001`/`1002` mean the token is
  unsupported on the destination/source chain; `1003` is a withdraw-init failure.
- **No idempotency key** is supported — do not assume safe blind retries on writes.
- **Pagination** is cursor-based via `next_page_token` / `page_size`.
- **Refunds/withdrawals** (`WithdrawLiquidity`) require a wallet-signed
  `withdraw_req` + `sig`; there is no server-issued API credential.
