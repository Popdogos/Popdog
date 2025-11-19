# x402 SDK Quick Start Guide

## Introduction

This guide provides a quick start for integrating x402 payment functionality into the Popdog website using the Rapid402 SDK for Solana.

## Prerequisites

- Node.js 14+ or npm/yarn
- Solana wallet (Phantom or Solflare)
- Basic knowledge of JavaScript/TypeScript
- Solana Web3.js understanding

## Installation

### Using npm

```bash
npm install @rapid402/sdk @solana/web3.js
```

### Using yarn

```bash
yarn add @rapid402/sdk @solana/web3.js
```

### CDN (Browser)

```html
<script src="https://unpkg.com/@rapid402/sdk/dist/index.umd.js"></script>
<script src="https://unpkg.com/@solana/web3.js@latest/lib/index.iife.js"></script>
```

## Basic Setup

### 1. Import SDK

```javascript
import { Rapid402Client } from '@rapid402/sdk';
import { Connection, PublicKey } from '@solana/web3.js';
```

### 2. Initialize Client

```javascript
const x402Client = new Rapid402Client({
    network: 'mainnet-beta', // or 'devnet' for testing
    rpcUrl: 'https://api.mainnet-beta.solana.com',
    // Optional: custom RPC endpoint
    // rpcUrl: 'https://your-custom-rpc-endpoint.com'
});
```

### 3. Configure Payment Parameters

```javascript
const PAYMENT_CONFIG = {
    // PopDog token mint address
    tokenMint: 'BpHUVW5Hbm2M9g1U6H8QRoCs1PGoDrSqWW3Cs4w2pump',
    
    // Payment amount (1 POPDOG = 1,000,000 if 6 decimals)
    amount: 1000000,
    
    // Token decimals
    decimals: 6,
    
    // Merchant wallet address (where payments go)
    merchantAddress: 'YOUR_MERCHANT_WALLET_ADDRESS',
    
    // Payment memo
    memo: 'Popdog x402 Payment'
};
```

## Implementation Example

### Complete Payment Flow

```javascript
// Initialize x402 client
const x402Client = new Rapid402Client({
    network: 'mainnet-beta',
    rpcUrl: 'https://api.mainnet-beta.solana.com'
});

// Payment configuration
const config = {
    tokenMint: 'BpHUVW5Hbm2M9g1U6H8QRoCs1PGoDrSqWW3Cs4w2pump',
    amount: 1000000,
    merchantAddress: 'YOUR_MERCHANT_WALLET',
    memo: 'Popdog Premium Access'
};

// Main payment function
async function processX402Payment() {
    try {
        // Step 1: Check wallet connection
        if (!window.solana || !window.solana.isConnected) {
            const response = await window.solana.connect();
            console.log('Connected:', response.publicKey.toString());
        }

        const wallet = window.solana;
        const payerPublicKey = wallet.publicKey;

        // Step 2: Create payment request
        console.log('Creating payment request...');
        const paymentRequest = await x402Client.createPaymentRequest({
            amount: config.amount,
            token: config.tokenMint,
            recipient: config.merchantAddress,
            memo: config.memo,
            payer: payerPublicKey.toString()
        });

        // Step 3: Sign and send transaction
        console.log('Signing transaction...');
        const signedTransaction = await wallet.signTransaction(
            paymentRequest.transaction
        );

        // Step 4: Submit payment
        console.log('Submitting payment...');
        const result = await x402Client.submitPayment({
            transaction: signedTransaction,
            signature: signedTransaction.signature
        });

        // Step 5: Verify payment
        if (result.success) {
            console.log('Payment successful!', result.signature);
            
            // Verify on-chain
            const verification = await x402Client.verifyPayment(
                result.signature
            );
            
            if (verification.verified) {
                // Grant access
                grantContentAccess();
                showSuccessMessage('Payment successful!');
            }
        } else {
            throw new Error('Payment failed');
        }

    } catch (error) {
        console.error('Payment error:', error);
        handlePaymentError(error);
    }
}

// Helper functions
function grantContentAccess() {
    // Unlock premium content
    document.getElementById('premiumContent').style.display = 'block';
    localStorage.setItem('popdog_paid', 'true');
}

function showSuccessMessage(message) {
    const toast = document.getElementById('liveMessage');
    toast.textContent = message;
    toast.style.display = 'block';
    setTimeout(() => {
        toast.style.display = 'none';
    }, 5000);
}

function handlePaymentError(error) {
    let message = 'Payment failed. Please try again.';
    
    if (error.code === 'WALLET_NOT_CONNECTED') {
        message = 'Please connect your wallet first.';
    } else if (error.code === 'INSUFFICIENT_FUNDS') {
        message = 'Insufficient funds. Please add more POPDOG.';
    } else if (error.code === 'USER_REJECTED') {
        message = 'Payment cancelled.';
    }
    
    alert(message);
}
```

## Integration Patterns

### Service Integration

Integrate x402 SDK into your payment service:

```typescript
import { Rapid402Client } from '@rapid402/sdk';
import { PaymentService } from './services/payment-service';

class PaymentServiceIntegration {
    private x402Client: Rapid402Client;
    
    constructor() {
        this.x402Client = new Rapid402Client({
            network: 'mainnet-beta',
            rpcUrl: process.env.SOLANA_RPC_URL
        });
    }
    
    async processPayment(paymentRequest: PaymentRequest): Promise<PaymentResult> {
        return await this.x402Client.createPaymentRequest(paymentRequest);
    }
}
```

### Event-Driven Integration

For event-driven architectures:

```typescript
import { EventEmitter } from 'events';
import { Rapid402Client } from '@rapid402/sdk';

class PaymentEventEmitter extends EventEmitter {
    private x402Client: Rapid402Client;
    
    constructor() {
        super();
        this.x402Client = new Rapid402Client({
            network: 'mainnet-beta'
        });
    }
    
    async initiatePayment(payment: Payment): Promise<void> {
        this.emit('payment.initiated', payment);
        
        try {
            const result = await this.x402Client.processPayment(payment);
            this.emit('payment.completed', result);
        } catch (error) {
            this.emit('payment.failed', error);
        }
    }
}
```

## Advanced Features

### Payment Status Tracking

```javascript
async function trackPaymentStatus(signature) {
    const status = await x402Client.getPaymentStatus(signature);
    
    switch (status.state) {
        case 'pending':
            console.log('Payment pending...');
            break;
        case 'confirmed':
            console.log('Payment confirmed!');
            grantContentAccess();
            break;
        case 'failed':
            console.log('Payment failed');
            break;
    }
    
    return status;
}
```

### Batch Payments

```javascript
async function processBatchPayments(payments) {
    const results = await Promise.all(
        payments.map(payment => x402Client.createPaymentRequest(payment))
    );
    
    // Process all payments
    return results;
}
```

### Payment History

```javascript
async function getPaymentHistory(walletAddress) {
    const history = await x402Client.getPaymentHistory({
        wallet: walletAddress,
        limit: 10
    });
    
    return history;
}
```

## Error Handling

### Common Errors

```javascript
const ERROR_CODES = {
    WALLET_NOT_CONNECTED: 'WALLET_NOT_CONNECTED',
    INSUFFICIENT_FUNDS: 'INSUFFICIENT_FUNDS',
    USER_REJECTED: 'USER_REJECTED',
    NETWORK_ERROR: 'NETWORK_ERROR',
    INVALID_TOKEN: 'INVALID_TOKEN',
    TRANSACTION_FAILED: 'TRANSACTION_FAILED'
};

function handleError(error) {
    switch (error.code) {
        case ERROR_CODES.WALLET_NOT_CONNECTED:
            // Prompt user to connect wallet
            connectWallet();
            break;
            
        case ERROR_CODES.INSUFFICIENT_FUNDS:
            // Show insufficient funds message
            showMessage('Insufficient POPDOG balance');
            break;
            
        case ERROR_CODES.USER_REJECTED:
            // User cancelled transaction
            showMessage('Payment cancelled');
            break;
            
        default:
            // Generic error handling
            console.error('Payment error:', error);
            showMessage('Payment failed. Please try again.');
    }
}
```

## Testing

### Testnet Setup

```javascript
// Use devnet for testing
const x402Client = new Rapid402Client({
    network: 'devnet',
    rpcUrl: 'https://api.devnet.solana.com'
});

// Get test tokens from Solana faucet
// https://faucet.solana.com/
```

### Test Payment Flow

```javascript
async function testPayment() {
    // Use test wallet
    const testWallet = await createTestWallet();
    
    // Process test payment
    await processX402Payment(testWallet);
    
    // Verify payment
    const verified = await verifyPayment(signature);
    console.log('Payment verified:', verified);
}
```

## Production Checklist

- [ ] Set correct network (mainnet-beta)
- [ ] Configure merchant wallet address
- [ ] Set correct token mint address
- [ ] Implement server-side verification
- [ ] Add error handling
- [ ] Test payment flow
- [ ] Add loading states
- [ ] Implement payment history
- [ ] Add analytics tracking
- [ ] Set up monitoring

## Troubleshooting

### Wallet Not Detected

```javascript
if (typeof window.solana === 'undefined') {
    alert('Please install Phantom wallet');
    window.open('https://phantom.app/', '_blank');
}
```

### Transaction Timeout

```javascript
const paymentPromise = processX402Payment();
const timeoutPromise = new Promise((_, reject) => 
    setTimeout(() => reject(new Error('Timeout')), 300000)
);

try {
    await Promise.race([paymentPromise, timeoutPromise]);
} catch (error) {
    if (error.message === 'Timeout') {
        showMessage('Payment timed out. Please try again.');
    }
}
```

### RPC Rate Limiting

```javascript
// Use custom RPC endpoint
const x402Client = new Rapid402Client({
    network: 'mainnet-beta',
    rpcUrl: 'https://your-custom-rpc.com' // Use paid RPC for production
});
```

## Resources

- **Rapid402 SDK**: https://github.com/rapid402/rapid402-sdk
- **Solana Web3.js**: https://solana-labs.github.io/solana-web3.js/
- **Solana Docs**: https://docs.solana.com/
- **Phantom Wallet**: https://phantom.app/
- **Solflare Wallet**: https://solflare.com/

## Support

For issues or questions:
- Check SDK documentation
- Open GitHub issue
- Join Solana Discord
- Contact Popdog team

---

**Version**: 1.0.0
**Last Updated**: January 2025

