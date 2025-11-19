# x402 Payment System Integration

## Overview

x402 is a protocol that enables seamless, on-chain micropayments over HTTP, facilitating monetization of APIs, websites, and autonomous agents. The Popdog website integrates x402 payment functionality to accept PopDog token payments on the Solana blockchain.

## What is x402?

x402 is an open protocol standard (RFC 402) that allows:
- **Micropayments**: Small, on-chain payments for content access
- **HTTP Integration**: Seamless payment flow over standard HTTP requests
- **Multi-Chain Support**: Works across various blockchain networks
- **API Monetization**: Enable pay-per-use API access
- **Content Gating**: Restrict access to premium content until payment

## Architecture

### Payment Flow

```
User → Wallet Connection → Payment Request → Transaction Signing → Payment Verification → Content Access
```

### Components

1. **Payment Card UI**: User-facing payment interface
2. **Wallet Integration**: Solana wallet connection (Phantom/Solflare)
3. **Transaction Handler**: Processes payment transactions
4. **Verification System**: Validates payment completion
5. **Content Gating**: Controls access based on payment status

## Current Implementation

### HTML Structure

```html
<div class="x402-payment-card">
    <div class="payment-header">
        <div class="wallet-icon">💳</div>
        <h3 class="payment-title">x402 Popdog Payment</h3>
        <p class="payment-subtitle">To access this content, please pay 1 POPDOG.</p>
    </div>
    <div class="payment-details">
        <!-- Wallet connection status -->
        <!-- Network information -->
        <!-- Payment amount -->
    </div>
    <button class="pay-now-btn" id="payNowBtn">Pay now</button>
</div>
```

### JavaScript Implementation

The current implementation includes:
- Wallet detection and connection
- Payment button handler
- Transaction status display
- Error handling

## SDK Integration Options

### Option 1: Rapid402 SDK (Recommended for Solana)

**GitHub**: [rapid402/rapid402-sdk](https://github.com/rapid402/rapid402-sdk)

**Installation**:
```bash
npm install @rapid402/sdk
# or
yarn add @rapid402/sdk
```

**Basic Usage**:
```javascript
import { Rapid402Client } from '@rapid402/sdk';

const client = new Rapid402Client({
    network: 'mainnet-beta', // or 'devnet' for testing
    rpcUrl: 'https://api.mainnet-beta.solana.com'
});

// Create payment request
const paymentRequest = await client.createPaymentRequest({
    amount: 1000000, // Amount in smallest unit (lamports)
    token: 'POPDOG_TOKEN_ADDRESS',
    recipient: 'MERCHANT_WALLET_ADDRESS',
    memo: 'Popdog payment'
});

// Process payment
const transaction = await client.processPayment(paymentRequest, wallet);
```

### Option 2: xPlug SDK for Solana

**Documentation**: [xPlug Documentation](https://docs.xplug.io/)

**Installation**:
```bash
npm install @xplug/sdk
```

**Basic Usage**:
```javascript
import { XPlug } from '@xplug/sdk';

const xplug = new XPlug({
    network: 'mainnet-beta'
});

// Initialize payment
const payment = await xplug.createPayment({
    amount: 1, // POPDOG amount
    token: 'POPDOG_TOKEN_MINT',
    recipient: 'MERCHANT_ADDRESS'
});

// Execute payment
await xplug.executePayment(payment, wallet);
```

### Option 3: Coinbase x402 Reference Implementation

**GitHub**: [coinbase/x402](https://github.com/coinbase/x402)

This is the reference implementation that other SDKs build upon.

## Integration Guide

### Step 1: Install SDK

Choose one of the SDKs above and install it:

```bash
npm install @rapid402/sdk
```

### Step 2: Initialize SDK

```javascript
import { Rapid402Client } from '@rapid402/sdk';

const x402Client = new Rapid402Client({
    network: 'mainnet-beta',
    rpcUrl: process.env.SOLANA_RPC_URL || 'https://api.mainnet-beta.solana.com'
});
```

### Step 3: Update Payment Handler

Replace the current `payNowBtn` handler with x402 SDK integration:

```javascript
async function processX402Payment() {
    try {
        // Check wallet connection
        if (!window.solana || !window.solana.isConnected) {
            await connectWallet();
        }

        const wallet = window.solana;
        const publicKey = wallet.publicKey.toString();

        // Create payment request
        const paymentRequest = await x402Client.createPaymentRequest({
            amount: 1000000, // 1 POPDOG (adjust based on token decimals)
            token: 'POPDOG_TOKEN_MINT_ADDRESS',
            recipient: 'MERCHANT_WALLET_ADDRESS',
            memo: 'Popdog x402 payment',
            metadata: {
                contentId: 'popdog-premium-content',
                timestamp: Date.now()
            }
        });

        // Request payment approval from wallet
        const signature = await wallet.signTransaction(paymentRequest.transaction);

        // Submit payment
        const result = await x402Client.submitPayment({
            transaction: paymentRequest.transaction,
            signature: signature
        });

        if (result.success) {
            // Payment successful
            showSuccessMessage('Payment successful! Access granted.');
            grantContentAccess();
        } else {
            throw new Error('Payment failed');
        }
    } catch (error) {
        console.error('Payment error:', error);
        showErrorMessage('Payment failed. Please try again.');
    }
}
```

### Step 4: Payment Verification

Implement server-side verification (recommended):

```javascript
// Server-side verification endpoint
async function verifyPayment(signature, publicKey) {
    const connection = new Connection('https://api.mainnet-beta.solana.com');
    const transaction = await connection.getTransaction(signature);
    
    // Verify transaction details
    // Check recipient, amount, token, etc.
    
    return {
        verified: true,
        amount: transaction.meta.postBalances[0],
        timestamp: transaction.blockTime
    };
}
```

## Configuration

### Environment Variables

Create a `.env` file:

```env
SOLANA_NETWORK=mainnet-beta
SOLANA_RPC_URL=https://api.mainnet-beta.solana.com
POPDOG_TOKEN_MINT=BpHUVW5Hbm2M9g1U6H8QRoCs1PGoDrSqWW3Cs4w2pump
MERCHANT_WALLET_ADDRESS=YOUR_MERCHANT_WALLET
PAYMENT_AMOUNT=1000000
```

### Payment Settings

```javascript
const PAYMENT_CONFIG = {
    tokenMint: 'BpHUVW5Hbm2M9g1U6H8QRoCs1PGoDrSqWW3Cs4w2pump', // PopDog token
    amount: 1000000, // Amount in smallest unit
    decimals: 6, // Token decimals
    network: 'mainnet-beta',
    merchantAddress: 'YOUR_MERCHANT_WALLET_ADDRESS',
    timeout: 300000 // 5 minutes
};
```

## API Reference

### Rapid402Client Methods

#### `createPaymentRequest(options)`
Creates a new payment request.

**Parameters**:
- `amount` (number): Payment amount in smallest unit
- `token` (string): Token mint address
- `recipient` (string): Recipient wallet address
- `memo` (string, optional): Payment memo
- `metadata` (object, optional): Additional metadata

**Returns**: `Promise<PaymentRequest>`

#### `processPayment(paymentRequest, wallet)`
Processes a payment using the connected wallet.

**Parameters**:
- `paymentRequest` (PaymentRequest): Payment request object
- `wallet` (Wallet): Connected Solana wallet

**Returns**: `Promise<PaymentResult>`

#### `verifyPayment(signature)`
Verifies a payment transaction.

**Parameters**:
- `signature` (string): Transaction signature

**Returns**: `Promise<VerificationResult>`

## Error Handling

```javascript
try {
    await processX402Payment();
} catch (error) {
    if (error.code === 'WALLET_NOT_CONNECTED') {
        // Handle wallet not connected
    } else if (error.code === 'INSUFFICIENT_FUNDS') {
        // Handle insufficient funds
    } else if (error.code === 'USER_REJECTED') {
        // Handle user rejection
    } else {
        // Handle other errors
    }
}
```

## Security Considerations

1. **Server-Side Verification**: Always verify payments on the server
2. **Transaction Signing**: Never expose private keys
3. **Amount Validation**: Verify payment amounts match expected values
4. **Replay Prevention**: Use nonces or timestamps to prevent replay attacks
5. **Rate Limiting**: Implement rate limiting on payment endpoints

## Testing

### Testnet Testing

```javascript
const x402Client = new Rapid402Client({
    network: 'devnet',
    rpcUrl: 'https://api.devnet.solana.com'
});
```

### Test Scenarios

1. **Successful Payment**: Complete payment flow
2. **Insufficient Funds**: Handle insufficient balance
3. **Wallet Rejection**: Handle user cancellation
4. **Network Errors**: Handle network failures
5. **Timeout**: Handle payment timeouts

## Best Practices

1. **User Experience**:
   - Show clear payment amounts
   - Display transaction status
   - Provide error messages
   - Offer retry options

2. **Performance**:
   - Cache payment requests
   - Optimize RPC calls
   - Use connection pooling

3. **Reliability**:
   - Implement retry logic
   - Handle edge cases
   - Monitor payment success rates

## Resources

- **x402 Protocol**: [RFC 402](https://datatracker.ietf.org/doc/html/rfc402)
- **Rapid402 SDK**: [GitHub](https://github.com/rapid402/rapid402-sdk)
- **xPlug Documentation**: [docs.xplug.io](https://docs.xplug.io/)
- **Solana Web3.js**: [Documentation](https://solana-labs.github.io/solana-web3.js/)
- **Coinbase x402**: [GitHub](https://github.com/coinbase/x402)

## Support

For issues or questions:
- Open an issue on GitHub
- Check SDK documentation
- Join Solana developer community

---

**Last Updated**: January 2025
**Version**: 1.0.0

