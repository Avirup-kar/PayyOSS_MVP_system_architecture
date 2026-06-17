# PayyOSS_MVP_system_architecture


# Payment Processing Flow

## Architecture Overview

```text
Merchant
    │
    ▼
Payment Intent (pi_xxx)
    │
    ▼
Checkout Session (cs_xxx)
    │
    ▼
Checkout Page
    │
    ▼
Wallet Connection
    │
    ▼
Smart Contract Payment
    │
    ▼
Blockchain Confirmation
    │
    ▼
Gateway Verification
    │
    ▼
Success / Cancel Redirect
```

---

# Database Relationships

```text
Merchant
    │
    │ 1:N
    ▼
Payment Intent
    │
    │ 1:1
    ▼
Checkout Session
    │
    │ 1:N
    ▼
Payments
```

---

# Entity Responsibilities

## Merchant

Stores merchant configuration.

```ts
Merchant {
  id
  walletAddress
  businessName
}
```

---

## Payment Intent

Represents what the customer must pay.

```ts
PaymentIntent {
  id
  merchantId
  amount
  amountRaw
  chainId
  tokenAddress
  tokenSymbol
  tokenDecimals
  status
  expiresAt
}
```

Example:

```json
{
  "id": "pi_123",
  "amount": "10",
  "chainId": 84532,
  "tokenAddress": "0xUSDC"
}
```

---

## Checkout Session

Represents a customer checkout experience.

```ts
CheckoutSession {
  id
  paymentIntentId
  successUrl
  cancelUrl
  status
}
```

Example:

```json
{
  "id": "cs_123",
  "paymentIntentId": "pi_123",
  "successUrl": "https://merchant.com/success",
  "cancelUrl": "https://merchant.com/cancel"
}
```

---

## Payments

Represents actual blockchain transactions.

```ts
Payment {
  id
  paymentIntentId
  payerWallet
  txHash
  status
  confirmedAt
}
```

---

# Checkout Flow

## 1. Merchant Creates Payment

```http
POST /payment-intents
```

Gateway creates:

```text
pi_123
```

---

## 2. Gateway Creates Checkout Session

```text
cs_123
```

Linked to:

```text
pi_123
```

---

## 3. Customer Opens Checkout

```text
https://checkout.gateway.com/cs_123
```

Frontend requests:

```http
GET /checkout-sessions/cs_123
```

Response:

```json
{
  "paymentIntentId": "pi_123",
  "amount": "10",
  "amountRaw": "10000000",

  "chainId": 84532,

  "tokenAddress": "0xUSDC",
  "tokenSymbol": "USDC",
  "tokenDecimals": 6,

  "merchantWallet": "0xMerchant",

  "successUrl": "https://merchant.com/success",
  "cancelUrl": "https://merchant.com/cancel"
}
```

---

## 4. Customer Connects Wallet

Supported:

- MetaMask
- Rabby
- Coinbase Wallet
- WalletConnect wallets

Frontend verifies:

```ts
walletChainId === paymentIntent.chainId
```

If mismatch:

```ts
await switchNetwork(paymentIntent.chainId);
```

---

## 5. Customer Clicks Pay

Frontend calls smart contract:

```solidity
pay(
    paymentIntentId,
    merchantWallet,
    amountRaw
)
```

---

## 6. Wallet Approval

### Approve

Transaction submitted.

Returns:

```text
txHash
```

Example:

```text
0xabc123...
```

### Reject

Wallet returns error.

Example:

```json
{
  "code": 4001,
  "message": "User rejected request"
}
```

Result:

```text
Payment Cancelled
Redirect -> cancelUrl
```

---

## 7. Submit Transaction To Gateway

Frontend sends:

```http
POST /payments/submit
```

```json
{
  "paymentIntentId": "pi_123",
  "txHash": "0xabc123..."
}
```

Gateway updates:

```text
payments.status = submitted
```

---

## 8. Blockchain Confirmation

Smart contract emits:

```solidity
event PaymentReceived(
    string paymentIntentId,
    address payer,
    address merchant,
    uint256 amount
);
```

---

## 9. Event Listener

NestJS listener receives event.

```text
Smart Contract
      │
      ▼
PaymentReceived Event
      │
      ▼
NestJS Listener
      │
      ▼
Database Update
```

Updates:

```text
payments.status = confirmed

payment_intents.status = succeeded
```

---

## 10. Frontend Polling

Frontend periodically requests:

```http
GET /payments/status/pi_123
```

Possible responses:

### Pending

```json
{
  "status": "pending"
}
```

### Confirmed

```json
{
  "status": "confirmed"
}
```

### Failed

```json
{
  "status": "failed"
}
```

---

## 11. Redirect Logic

```text
Confirmed
    │
    ▼
successUrl
```

```text
Failed
    │
    ▼
cancelUrl
```

```text
Expired
    │
    ▼
cancelUrl
```

---

# Sandbox Environment

```text
Environment: Sandbox

Chain ID: 84532
Token: Test USDC
Contract: Test Contract
```

Customer uses:

- Test ETH
- Test USDC

No real funds move.

---

# Production Environment

```text
Environment: Live

Chain ID: 8453
Token: USDC
Contract: Production Contract
```

Customer uses:

- Real ETH
- Real USDC

Real funds move.

---

# Final Production Flow

```text
Merchant
    │
    ▼
Create Payment Intent
    │
    ▼
Create Checkout Session
    │
    ▼
Customer Opens Checkout
    │
    ▼
Connect Wallet
    │
    ▼
Network Validation
    │
    ▼
Approve Transaction
    │
    ▼
Smart Contract
    │
    ▼
PaymentReceived Event
    │
    ▼
NestJS Listener
    │
    ▼
Update Payment Status
    │
    ▼
Frontend Polling
    │
    ▼
Success / Cancel Redirect
```
