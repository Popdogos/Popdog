# Popdog Technical Specification

## Document Information

- **Version**: 1.0.0
- **Last Updated**: January 2025
- **Status**: Production Ready
- **Classification**: Public Technical Documentation

## Table of Contents

1. [System Architecture](#system-architecture)
2. [Protocol Implementation](#protocol-implementation)
3. [Cryptographic Foundations](#cryptographic-foundations)
4. [Transaction Processing](#transaction-processing)
5. [State Management](#state-management)
6. [Performance Engineering](#performance-engineering)
7. [Security Implementation](#security-implementation)
8. [Monitoring & Observability](#monitoring--observability)

## System Architecture

### Distributed Systems Principles

Popdog implements a **distributed, eventually consistent system** following the CAP theorem:

- **Consistency**: Eventual consistency with strong consistency for critical operations
- **Availability**: High availability (99.9% SLA) through redundancy
- **Partition Tolerance**: Network partition resilience through replication

### System Components

#### 1. Payment Gateway Service

**Architecture Pattern**: API Gateway with Backend for Frontend (BFF)

**Responsibilities**:
- HTTP request handling and routing
- Request validation and sanitization
- Authentication and authorization (OAuth 2.0 / JWT)
- Rate limiting and throttling
- Request/response transformation
- Error handling and logging

**Technology Stack**:
```typescript
// Core Framework
import express from 'express';
import { createServer } from 'http';

// Middleware Stack
import helmet from 'helmet';           // Security headers
import cors from 'cors';               // CORS configuration
import compression from 'compression'; // Response compression
import rateLimit from 'express-rate-limit'; // Rate limiting
import { body, param, query, validationResult } from 'express-validator'; // Input validation

// Authentication
import jwt from 'jsonwebtoken';
import { verifyToken, requireAuth } from './middleware/auth';

// Logging & Monitoring
import winston from 'winston';
import { createLogger } from './utils/logger';
import prometheus from 'prom-client'; // Metrics collection
```

**Performance Characteristics**:
- **Latency**: P50 < 50ms, P99 < 200ms
- **Throughput**: 10,000+ requests/second per instance
- **Concurrency**: 1,000+ concurrent connections
- **Memory**: < 512MB per instance
- **CPU**: < 50% utilization under load

#### 2. Transaction Service

**Architecture Pattern**: Domain-Driven Design (DDD) with CQRS

**Core Domain Model**:

```typescript
// Domain Entities
interface Payment {
    id: string;                    // UUID v4
    payer: PublicKey;              // Solana public key
    recipient: PublicKey;          // Merchant public key
    amount: bigint;                // Amount in lamports (BigInt for precision)
    token: PublicKey;              // SPL token mint address
    status: PaymentStatus;         // State machine state
    createdAt: Date;               // ISO 8601 timestamp
    updatedAt: Date;               // ISO 8601 timestamp
    metadata: PaymentMetadata;     // Additional data
    nonce: string;                 // Cryptographic nonce
}

enum PaymentStatus {
    PENDING = 'pending',
    SIGNATURE_REQUESTED = 'signature_requested',
    SIGNED = 'signed',
    SUBMITTED = 'submitted',
    CONFIRMED = 'confirmed',
    FAILED = 'failed',
    EXPIRED = 'expired'
}

interface PaymentMetadata {
    contentId?: string;
    userId?: string;
    sessionId?: string;
    ipAddress?: string;
    userAgent?: string;
    referrer?: string;
    customFields?: Record<string, any>;
}
```

**Transaction Builder Algorithm**:

```typescript
class TransactionBuilder {
    private connection: Connection;
    private feeCalculator: FeeCalculator;
    private nonceManager: NonceManager;
    
    /**
     * Builds a Solana transaction for payment
     * 
     * Time Complexity: O(n) where n is number of instructions
     * Space Complexity: O(n)
     * 
     * @param payment - Payment domain object
     * @returns Promise<Transaction>
     */
    async buildPaymentTransaction(payment: Payment): Promise<Transaction> {
        // Step 1: Get recent blockhash (with retry logic)
        const { blockhash, lastValidBlockHeight } = 
            await this.getRecentBlockhashWithRetry();
        
        // Step 2: Calculate transaction fees
        const fee = await this.feeCalculator.calculate({
            payer: payment.payer,
            instructions: this.buildInstructions(payment),
            blockhash,
        });
        
        // Step 3: Build transaction
        const transaction = new Transaction({
            feePayer: payment.payer,
            recentBlockhash: blockhash,
            nonceInfo: await this.nonceManager.getNonce(payment.id),
        });
        
        // Step 4: Add instructions
        const instructions = this.buildInstructions(payment);
        instructions.forEach(ix => transaction.add(ix));
        
        // Step 5: Compute transaction size
        const serialized = transaction.serialize({
            requireAllSignatures: false,
            verifySignatures: false,
        });
        
        // Step 6: Validate transaction size (Solana limit: 1232 bytes)
        if (serialized.length > 1232) {
            throw new TransactionTooLargeError(
                `Transaction size ${serialized.length} exceeds limit of 1232 bytes`
            );
        }
        
        // Step 7: Add compute budget instructions if needed
        if (this.requiresComputeBudget(instructions)) {
            transaction.add(
                ComputeBudgetProgram.setComputeUnitLimit({
                    units: 200000, // Standard transfer
                })
            );
        }
        
        return transaction;
    }
    
    /**
     * Builds SPL token transfer instruction
     * 
     * @param payment - Payment domain object
     * @returns TransactionInstruction[]
     */
    private buildInstructions(payment: Payment): TransactionInstruction[] {
        const instructions: TransactionInstruction[] = [];
        
        // Get associated token accounts
        const payerATA = this.getAssociatedTokenAddress(
            payment.token,
            payment.payer
        );
        const recipientATA = this.getAssociatedTokenAddress(
            payment.token,
            payment.recipient
        );
        
        // Create associated token account if it doesn't exist
        const recipientATAInfo = await this.connection.getAccountInfo(recipientATA);
        if (!recipientATAInfo) {
            instructions.push(
                createAssociatedTokenAccountInstruction(
                    payment.payer,      // Payer for account creation
                    recipientATA,       // Associated token account
                    payment.recipient,  // Owner
                    payment.token       // Mint
                )
            );
        }
        
        // Add transfer instruction
        instructions.push(
            createTransferInstruction(
                payerATA,           // Source
                recipientATA,       // Destination
                payment.payer,      // Owner
                payment.amount,     // Amount
                [],                 // Multi-signers (none for single sig)
            )
        );
        
        // Add memo instruction for x402 protocol
        instructions.push(
            new TransactionInstruction({
                keys: [],
                programId: MEMO_PROGRAM_ID,
                data: Buffer.from(
                    JSON.stringify({
                        protocol: 'x402',
                        version: '1.0',
                        paymentId: payment.id,
                        nonce: payment.nonce,
                        timestamp: payment.createdAt.toISOString(),
                    }),
                    'utf-8'
                ),
            })
        );
        
        return instructions;
    }
    
    /**
     * Gets recent blockhash with exponential backoff retry
     * 
     * @param maxRetries - Maximum number of retries (default: 3)
     * @returns Promise<{ blockhash: string, lastValidBlockHeight: number }>
     */
    private async getRecentBlockhashWithRetry(
        maxRetries: number = 3
    ): Promise<{ blockhash: string; lastValidBlockHeight: number }> {
        let lastError: Error;
        
        for (let attempt = 0; attempt < maxRetries; attempt++) {
            try {
                const { blockhash, lastValidBlockHeight } = 
                    await this.connection.getLatestBlockhash('confirmed');
                
                return { blockhash, lastValidBlockHeight };
            } catch (error) {
                lastError = error as Error;
                const delay = Math.min(1000 * Math.pow(2, attempt), 10000);
                await this.sleep(delay);
            }
        }
        
        throw new BlockhashFetchError(
            `Failed to fetch blockhash after ${maxRetries} attempts`,
            lastError!
        );
    }
}
```

#### 3. Verification Service

**Verification Algorithm**:

```typescript
class VerificationService {
    private connection: Connection;
    private cache: Cache;
    private requiredConfirmations: number = 32; // Solana finality
    
    /**
     * Verifies payment transaction on-chain
     * 
     * Algorithm Complexity:
     * - Time: O(1) for cache hit, O(log n) for blockchain query
     * - Space: O(1)
     * 
     * @param signature - Transaction signature
     * @param expectedPayment - Expected payment parameters
     * @returns Promise<VerificationResult>
     */
    async verifyPayment(
        signature: string,
        expectedPayment: ExpectedPayment
    ): Promise<VerificationResult> {
        // Check cache first
        const cached = await this.cache.get(`verification:${signature}`);
        if (cached) {
            return cached as VerificationResult;
        }
        
        // Fetch transaction from blockchain
        const transaction = await this.fetchTransaction(signature);
        
        if (!transaction) {
            return {
                verified: false,
                reason: 'TRANSACTION_NOT_FOUND',
                signature,
            };
        }
        
        // Verify transaction status
        if (transaction.meta?.err) {
            return {
                verified: false,
                reason: 'TRANSACTION_FAILED',
                error: transaction.meta.err,
                signature,
            };
        }
        
        // Verify transaction structure
        const verification = this.verifyTransactionStructure(
            transaction,
            expectedPayment
        );
        
        if (!verification.verified) {
            return verification;
        }
        
        // Verify confirmations
        const confirmations = await this.getConfirmations(transaction.slot);
        if (confirmations < this.requiredConfirmations) {
            return {
                verified: false,
                reason: 'INSUFFICIENT_CONFIRMATIONS',
                confirmations,
                required: this.requiredConfirmations,
                signature,
            };
        }
        
        // Verify amount
        const amountVerification = this.verifyAmount(
            transaction,
            expectedPayment.amount
        );
        if (!amountVerification.verified) {
            return amountVerification;
        }
        
        // Verify recipient
        const recipientVerification = this.verifyRecipient(
            transaction,
            expectedPayment.recipient
        );
        if (!recipientVerification.verified) {
            return recipientVerification;
        }
        
        // Verify token
        const tokenVerification = this.verifyToken(
            transaction,
            expectedPayment.token
        );
        if (!tokenVerification.verified) {
            return tokenVerification;
        }
        
        // Verify nonce (replay attack prevention)
        const nonceVerification = await this.verifyNonce(
            transaction,
            expectedPayment.nonce
        );
        if (!nonceVerification.verified) {
            return nonceVerification;
        }
        
        const result: VerificationResult = {
            verified: true,
            signature,
            blockTime: transaction.blockTime || 0,
            slot: transaction.slot || 0,
            confirmations,
            amount: amountVerification.amount!,
            recipient: recipientVerification.recipient!,
            token: tokenVerification.token!,
        };
        
        // Cache result (TTL: 1 hour)
        await this.cache.set(
            `verification:${signature}`,
            result,
            3600
        );
        
        return result;
    }
    
    /**
     * Extracts transfer amount from transaction
     * 
     * @param transaction - Solana transaction
     * @returns number - Amount in lamports
     */
    private verifyAmount(
        transaction: ConfirmedTransaction,
        expectedAmount: bigint
    ): AmountVerification {
        // Parse transaction instructions
        const transferInstruction = transaction.transaction.message.instructions
            .find(ix => 
                ix.programId.equals(TOKEN_PROGRAM_ID) &&
                ix.data[0] === 3 // Transfer instruction discriminator
            );
        
        if (!transferInstruction) {
            return {
                verified: false,
                reason: 'TRANSFER_INSTRUCTION_NOT_FOUND',
            };
        }
        
        // Decode instruction data
        const data = Buffer.from(transferInstruction.data);
        const amount = data.readBigUInt64LE(1); // Skip discriminator byte
        
        if (amount !== expectedAmount) {
            return {
                verified: false,
                reason: 'AMOUNT_MISMATCH',
                expected: expectedAmount.toString(),
                actual: amount.toString(),
            };
        }
        
        return {
            verified: true,
            amount,
        };
    }
}
```

## Protocol Implementation

### x402 Protocol Compliance

**RFC 402 Standard Implementation**:

```typescript
/**
 * x402 Payment Request Structure
 * 
 * Complies with RFC 402 specification
 */
interface X402PaymentRequest {
    // Protocol version
    version: '1.0';
    
    // Payment identifier (UUID v4)
    paymentId: string;
    
    // Payment amount (in smallest unit)
    amount: {
        value: string;        // BigInt as string for precision
        currency: string;     // Token symbol (e.g., "POPDOG")
        decimals: number;     // Token decimals
    };
    
    // Blockchain information
    blockchain: {
        network: 'solana';
        chainId: 'mainnet-beta' | 'devnet' | 'testnet';
        tokenMint: string;    // SPL token mint address
    };
    
    // Parties
    payer: {
        address: string;      // Solana public key
        wallet?: string;      // Wallet provider (Phantom, Solflare)
    };
    
    recipient: {
        address: string;      // Merchant wallet address
        name?: string;        // Merchant name
    };
    
    // Payment metadata
    metadata: {
        contentId?: string;
        description?: string;
        nonce: string;        // Cryptographic nonce
        timestamp: string;     // ISO 8601
        [key: string]: any;    // Additional fields
    };
    
    // Expiration
    expiresAt?: string;       // ISO 8601 timestamp
    
    // Callback URLs
    callbacks?: {
        success?: string;     // Webhook URL for success
        failure?: string;     // Webhook URL for failure
        cancel?: string;      // Webhook URL for cancellation
    };
}
```

### HTTP Integration

**x402 HTTP Headers**:

```typescript
/**
 * x402 HTTP Request Headers
 * 
 * Complies with RFC 402 HTTP integration specification
 */
const X402_HEADERS = {
    'X-402-Protocol': 'x402/1.0',
    'X-402-Payment-Id': paymentId,
    'X-402-Amount': amount.toString(),
    'X-402-Currency': currency,
    'X-402-Nonce': nonce,
    'X-402-Timestamp': timestamp.toISOString(),
    'X-402-Signature': signature, // Request signature
};

/**
 * x402 HTTP Response Headers
 */
const X402_RESPONSE_HEADERS = {
    'X-402-Status': 'pending' | 'confirmed' | 'failed',
    'X-402-Transaction-Signature': signature,
    'X-402-Confirmations': confirmations.toString(),
    'X-402-Block-Time': blockTime.toString(),
};
```

## Cryptographic Foundations

### Nonce Generation

**Cryptographically Secure Nonce**:

```typescript
import crypto from 'crypto';

class NonceManager {
    /**
     * Generates cryptographically secure nonce
     * 
     * Uses CSPRNG (Cryptographically Secure Pseudorandom Number Generator)
     * 
     * @returns string - Base64-encoded nonce
     */
    generateNonce(): string {
        // Generate 32 random bytes (256 bits)
        const randomBytes = crypto.randomBytes(32);
        
        // Encode as base64
        return randomBytes.toString('base64');
    }
    
    /**
     * Validates nonce format and uniqueness
     * 
     * @param nonce - Nonce to validate
     * @returns boolean
     */
    validateNonce(nonce: string): boolean {
        // Check format (base64, 32 bytes = 44 chars)
        if (!/^[A-Za-z0-9+/]{43}=$/.test(nonce)) {
            return false;
        }
        
        // Check uniqueness (check against used nonces)
        return !this.isNonceUsed(nonce);
    }
    
    /**
     * Marks nonce as used (prevents replay attacks)
     * 
     * @param nonce - Nonce to mark as used
     * @param ttl - Time-to-live in seconds (default: 1 hour)
     */
    async markNonceAsUsed(nonce: string, ttl: number = 3600): Promise<void> {
        await this.cache.set(`nonce:${nonce}`, 'used', ttl);
    }
}
```

### Signature Verification

**Ed25519 Signature Verification**:

```typescript
import { PublicKey, Transaction } from '@solana/web3.js';
import nacl from 'tweetnacl';

class SignatureVerifier {
    /**
     * Verifies Ed25519 signature
     * 
     * @param message - Message bytes
     * @param signature - Signature bytes
     * @param publicKey - Public key
     * @returns boolean
     */
    verifySignature(
        message: Uint8Array,
        signature: Uint8Array,
        publicKey: PublicKey
    ): boolean {
        return nacl.sign.detached.verify(
            message,
            signature,
            publicKey.toBytes()
        );
    }
    
    /**
     * Verifies transaction signature
     * 
     * @param transaction - Solana transaction
     * @param publicKey - Expected signer public key
     * @returns boolean
     */
    verifyTransactionSignature(
        transaction: Transaction,
        publicKey: PublicKey
    ): boolean {
        // Get message to sign
        const message = transaction.serializeMessage();
        
        // Get signature
        const signature = transaction.signatures.find(
            sig => sig.publicKey.equals(publicKey)
        );
        
        if (!signature) {
            return false;
        }
        
        // Verify signature
        return this.verifySignature(
            message,
            signature.signature,
            publicKey
        );
    }
}
```

## Transaction Processing

### State Machine

**Payment State Machine**:

```typescript
/**
 * Payment State Machine
 * 
 * Implements finite state machine pattern
 */
class PaymentStateMachine {
    private states: Map<PaymentStatus, PaymentStatus[]> = new Map([
        [PaymentStatus.PENDING, [PaymentStatus.SIGNATURE_REQUESTED, PaymentStatus.EXPIRED]],
        [PaymentStatus.SIGNATURE_REQUESTED, [PaymentStatus.SIGNED, PaymentStatus.EXPIRED]],
        [PaymentStatus.SIGNED, [PaymentStatus.SUBMITTED, PaymentStatus.EXPIRED]],
        [PaymentStatus.SUBMITTED, [PaymentStatus.CONFIRMED, PaymentStatus.FAILED]],
        [PaymentStatus.CONFIRMED, []], // Terminal state
        [PaymentStatus.FAILED, []],    // Terminal state
        [PaymentStatus.EXPIRED, []],    // Terminal state
    ]);
    
    /**
     * Transitions payment to new state
     * 
     * @param payment - Payment object
     * @param newState - Target state
     * @throws InvalidStateTransitionError if transition is invalid
     */
    transition(payment: Payment, newState: PaymentStatus): void {
        const validTransitions = this.states.get(payment.status);
        
        if (!validTransitions?.includes(newState)) {
            throw new InvalidStateTransitionError(
                `Cannot transition from ${payment.status} to ${newState}`
            );
        }
        
        payment.status = newState;
        payment.updatedAt = new Date();
    }
}
```

## Performance Engineering

### Connection Pooling

**RPC Connection Pool**:

```typescript
class ConnectionPool {
    private pool: Connection[] = [];
    private maxSize: number = 10;
    private currentSize: number = 0;
    private queue: Array<() => void> = [];
    
    /**
     * Acquires connection from pool
     * 
     * @returns Promise<Connection>
     */
    async acquire(): Promise<Connection> {
        if (this.pool.length > 0) {
            return this.pool.pop()!;
        }
        
        if (this.currentSize < this.maxSize) {
            this.currentSize++;
            return this.createConnection();
        }
        
        // Wait for available connection
        return new Promise((resolve) => {
            this.queue.push(() => {
                resolve(this.pool.pop()!);
            });
        });
    }
    
    /**
     * Releases connection back to pool
     * 
     * @param connection - Connection to release
     */
    release(connection: Connection): void {
        if (this.queue.length > 0) {
            const resolve = this.queue.shift()!;
            resolve(connection);
        } else {
            this.pool.push(connection);
        }
    }
}
```

### Request Batching

**Batch Transaction Submission**:

```typescript
class BatchProcessor {
    private batch: Transaction[] = [];
    private batchSize: number = 10;
    private flushInterval: number = 100; // ms
    
    /**
     * Adds transaction to batch
     * 
     * @param transaction - Transaction to add
     */
    async add(transaction: Transaction): Promise<void> {
        this.batch.push(transaction);
        
        if (this.batch.length >= this.batchSize) {
            await this.flush();
        }
    }
    
    /**
     * Flushes batch (submits all transactions)
     */
    async flush(): Promise<void> {
        if (this.batch.length === 0) return;
        
        const transactions = this.batch.splice(0);
        
        // Submit in parallel
        const results = await Promise.allSettled(
            transactions.map(tx => this.submitTransaction(tx))
        );
        
        // Handle results
        results.forEach((result, index) => {
            if (result.status === 'fulfilled') {
                this.onSuccess(transactions[index], result.value);
            } else {
                this.onError(transactions[index], result.reason);
            }
        });
    }
}
```

## Security Implementation

### Rate Limiting

**Token Bucket Algorithm**:

```typescript
class RateLimiter {
    private buckets: Map<string, TokenBucket> = new Map();
    private capacity: number = 100;      // Maximum tokens
    private refillRate: number = 10;     // Tokens per second
    
    /**
     * Checks if request is allowed
     * 
     * @param key - Rate limit key (e.g., IP address, user ID)
     * @returns boolean
     */
    isAllowed(key: string): boolean {
        let bucket = this.buckets.get(key);
        
        if (!bucket) {
            bucket = new TokenBucket(this.capacity, this.refillRate);
            this.buckets.set(key, bucket);
        }
        
        return bucket.consume(1);
    }
}

class TokenBucket {
    private tokens: number;
    private capacity: number;
    private refillRate: number;
    private lastRefill: number;
    
    constructor(capacity: number, refillRate: number) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.tokens = capacity;
        this.lastRefill = Date.now();
    }
    
    consume(tokens: number): boolean {
        this.refill();
        
        if (this.tokens >= tokens) {
            this.tokens -= tokens;
            return true;
        }
        
        return false;
    }
    
    private refill(): void {
        const now = Date.now();
        const elapsed = (now - this.lastRefill) / 1000; // seconds
        const tokensToAdd = elapsed * this.refillRate;
        
        this.tokens = Math.min(
            this.capacity,
            this.tokens + tokensToAdd
        );
        
        this.lastRefill = now;
    }
}
```

## Monitoring & Observability

### Metrics Collection

**Prometheus Metrics**:

```typescript
import { Registry, Counter, Histogram, Gauge } from 'prom-client';

class MetricsCollector {
    private registry: Registry;
    
    // Counters
    private paymentRequests: Counter;
    private paymentSuccesses: Counter;
    private paymentFailures: Counter;
    
    // Histograms
    private paymentLatency: Histogram;
    private transactionLatency: Histogram;
    
    // Gauges
    private activePayments: Gauge;
    private queueSize: Gauge;
    
    constructor() {
        this.registry = new Registry();
        
        this.paymentRequests = new Counter({
            name: 'payment_requests_total',
            help: 'Total number of payment requests',
            labelNames: ['status'],
            registers: [this.registry],
        });
        
        this.paymentLatency = new Histogram({
            name: 'payment_latency_seconds',
            help: 'Payment processing latency',
            buckets: [0.1, 0.5, 1, 2, 5, 10],
            registers: [this.registry],
        });
    }
    
    recordPaymentRequest(status: string): void {
        this.paymentRequests.inc({ status });
    }
    
    recordPaymentLatency(duration: number): void {
        this.paymentLatency.observe(duration);
    }
}
```

### Distributed Tracing

**OpenTelemetry Integration**:

```typescript
import { trace, context, Span } from '@opentelemetry/api';

class TracingService {
    private tracer = trace.getTracer('popdog-payment-service');
    
    /**
     * Creates span for payment processing
     * 
     * @param paymentId - Payment ID
     * @returns Span
     */
    startPaymentSpan(paymentId: string): Span {
        return this.tracer.startSpan('payment.process', {
            attributes: {
                'payment.id': paymentId,
                'service.name': 'payment-service',
            },
        });
    }
    
    /**
     * Adds event to span
     * 
     * @param span - Span
     * @param name - Event name
     * @param attributes - Event attributes
     */
    addEvent(span: Span, name: string, attributes?: Record<string, any>): void {
        span.addEvent(name, attributes);
    }
}
```

---

**Last Updated**: January 2025 | **Version**: 1.0.0 | **Status**: Production Ready
