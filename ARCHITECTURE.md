# Popdog Architecture Specification

## Table of Contents

1. [System Overview](#system-overview)
2. [Architectural Patterns](#architectural-patterns)
3. [Component Architecture](#component-architecture)
4. [Data Flow Architecture](#data-flow-architecture)
5. [Security Architecture](#security-architecture)
6. [Scalability Architecture](#scalability-architecture)
7. [Deployment Architecture](#deployment-architecture)

## System Overview

### High-Level Architecture

Popdog implements a **distributed, event-driven microservices architecture** designed for high availability, scalability, and fault tolerance. The system is built on the following principles:

- **Separation of Concerns**: Each service has a single, well-defined responsibility
- **Event-Driven Communication**: Services communicate via events and message queues
- **Stateless Services**: Services maintain no local state, enabling horizontal scaling
- **API Gateway Pattern**: Single entry point for all client requests
- **Circuit Breaker Pattern**: Fault tolerance for external dependencies

### System Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Applications                       │
│              (Web, Mobile, API Clients, Autonomous Agents)      │
└────────────────────────────┬────────────────────────────────────┘
                             │ HTTPS/TLS 1.3
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                      API Gateway Layer                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Rate       │  │  Auth/       │  │   Load       │         │
│  │  Limiting    │  │  Authz       │  │  Balancing   │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Microservices Layer                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  Payment     │  │ Transaction  │  │ Verification │         │
│  │  Gateway     │  │  Service     │  │  Service     │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Notification│  │  Analytics   │  │  Wallet     │         │
│  │  Service     │  │  Service     │  │  Adapter     │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Message Queue Layer                          │
│         (RabbitMQ / Apache Kafka / AWS SQS)                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Blockchain Layer                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  Solana RPC  │  │  Transaction │  │  Validator   │         │
│  │  Endpoints   │  │  Builder     │  │  Network     │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

## Architectural Patterns

### 1. Microservices Architecture

Each service is independently deployable and scalable:

- **Payment Gateway Service**: Handles HTTP requests, validates input, initiates payment flow
- **Transaction Service**: Constructs Solana transactions, manages signing workflow
- **Verification Service**: Verifies transactions on-chain, maintains payment state
- **Notification Service**: Publishes payment events, sends webhooks/notifications
- **Analytics Service**: Collects metrics, generates reports, monitors performance

### 2. Event-Driven Architecture

Services communicate asynchronously via events:

```typescript
// Event Types
interface PaymentInitiatedEvent {
    paymentId: string;
    amount: number;
    token: string;
    payer: string;
    timestamp: number;
}

interface PaymentConfirmedEvent {
    paymentId: string;
    signature: string;
    blockTime: number;
    status: 'confirmed' | 'failed';
}

interface PaymentFailedEvent {
    paymentId: string;
    error: string;
    reason: string;
    timestamp: number;
}
```

### 3. CQRS (Command Query Responsibility Segregation)

Separate read and write models:

- **Write Model**: Optimized for transaction processing
- **Read Model**: Optimized for query performance (denormalized)
- **Event Sourcing**: All state changes stored as events

### 4. Saga Pattern

Distributed transaction management:

```typescript
class PaymentSaga {
    async execute(paymentRequest: PaymentRequest): Promise<PaymentResult> {
        // Step 1: Create payment record
        const payment = await this.createPayment(paymentRequest);
        
        // Step 2: Build transaction
        const transaction = await this.buildTransaction(payment);
        
        // Step 3: Request signature
        const signature = await this.requestSignature(transaction);
        
        // Step 4: Submit to blockchain
        const result = await this.submitTransaction(signature);
        
        // Step 5: Verify confirmation
        const verified = await this.verifyTransaction(result.signature);
        
        // Compensate on failure
        if (!verified) {
            await this.compensate(payment.id);
        }
        
        return result;
    }
}
```

## Component Architecture

### Payment Gateway Service

**Responsibilities**:
- HTTP endpoint management
- Request validation
- Authentication/authorization
- Rate limiting
- Request routing

**Technology Stack**:
- Node.js + Express/Fastify
- TypeScript for type safety
- OpenAPI/Swagger for API documentation
- JWT for authentication

**API Endpoints**:
```
POST   /api/v1/payments          Create payment request
GET    /api/v1/payments/:id       Get payment status
GET    /api/v1/payments           List payments (with pagination)
POST   /api/v1/payments/:id/verify Verify payment
```

### Transaction Service

**Responsibilities**:
- Transaction construction
- Fee calculation
- Nonce management
- Transaction signing workflow
- Transaction submission

**Key Algorithms**:

```typescript
class TransactionBuilder {
    async buildPaymentTransaction(
        payer: PublicKey,
        recipient: PublicKey,
        amount: number,
        token: PublicKey
    ): Promise<Transaction> {
        // Calculate fees dynamically
        const fee = await this.calculateFee(transactionSize);
        
        // Get recent blockhash
        const { blockhash } = await this.connection.getLatestBlockhash();
        
        // Build transaction
        const transaction = new Transaction({
            feePayer: payer,
            recentBlockhash: blockhash,
        });
        
        // Add token transfer instruction
        transaction.add(
            createTransferInstruction(
                payer,
                recipient,
                amount,
                token
            )
        );
        
        // Add memo instruction
        transaction.add(
            new TransactionInstruction({
                keys: [],
                programId: MEMO_PROGRAM_ID,
                data: Buffer.from('x402 payment', 'utf-8'),
            })
        );
        
        return transaction;
    }
    
    private async calculateFee(size: number): Promise<number> {
        // Dynamic fee calculation based on:
        // - Base fee (5000 lamports)
        // - Transaction size
        // - Network congestion
        const baseFee = 5000;
        const sizeFee = size * 100; // 100 lamports per byte
        const congestionMultiplier = await this.getCongestionMultiplier();
        
        return Math.ceil((baseFee + sizeFee) * congestionMultiplier);
    }
}
```

### Verification Service

**Responsibilities**:
- On-chain transaction verification
- Payment state management
- Confirmation monitoring
- Fraud detection
- Audit logging

**Verification Algorithm**:

```typescript
class VerificationService {
    async verifyPayment(
        signature: string,
        expectedAmount: number,
        expectedRecipient: PublicKey,
        expectedToken: PublicKey
    ): Promise<VerificationResult> {
        // Fetch transaction from blockchain
        const transaction = await this.connection.getTransaction(signature, {
            commitment: 'confirmed',
            maxSupportedTransactionVersion: 0,
        });
        
        if (!transaction) {
            return { verified: false, reason: 'Transaction not found' };
        }
        
        // Verify transaction status
        if (transaction.meta?.err) {
            return { verified: false, reason: 'Transaction failed' };
        }
        
        // Verify amount
        const actualAmount = this.extractTransferAmount(transaction);
        if (actualAmount !== expectedAmount) {
            return { verified: false, reason: 'Amount mismatch' };
        }
        
        // Verify recipient
        const actualRecipient = this.extractRecipient(transaction);
        if (!actualRecipient.equals(expectedRecipient)) {
            return { verified: false, reason: 'Recipient mismatch' };
        }
        
        // Verify token
        const actualToken = this.extractToken(transaction);
        if (!actualToken.equals(expectedToken)) {
            return { verified: false, reason: 'Token mismatch' };
        }
        
        // Verify confirmation
        const confirmations = transaction.slot ? 
            await this.getConfirmations(transaction.slot) : 0;
        
        if (confirmations < this.requiredConfirmations) {
            return { 
                verified: false, 
                reason: 'Insufficient confirmations',
                confirmations 
            };
        }
        
        return {
            verified: true,
            signature,
            blockTime: transaction.blockTime,
            confirmations,
        };
    }
}
```

## Data Flow Architecture

### Payment Flow Sequence

```
┌─────────┐      ┌──────────┐      ┌─────────────┐      ┌──────────┐
│ Client  │─────▶│  API     │─────▶│ Transaction │─────▶│ Solana   │
│         │      │ Gateway  │      │  Service    │      │ Network  │
└─────────┘      └──────────┘      └─────────────┘      └──────────┘
     │                 │                  │                    │
     │                 │                  │                    │
     │                 ▼                  │                    │
     │          ┌──────────┐             │                    │
     │          │ Payment  │             │                    │
     │          │ Created  │             │                    │
     │          └──────────┘             │                    │
     │                 │                  │                    │
     │                 ▼                  │                    │
     │          ┌──────────┐             │                    │
     │          │ Wallet   │             │                    │
     │          │ Signature│             │                    │
     │          │ Request  │             │                    │
     │          └──────────┘             │                    │
     │                 │                  │                    │
     │                 ▼                  │                    │
     │          ┌──────────┐             │                    │
     │          │ Signed   │             │                    │
     │          │ Transaction            │                    │
     │          └──────────┘             │                    │
     │                 │                  │                    │
     │                 ▼                  ▼                    │
     │          ┌──────────────────────────┐                 │
     │          │ Transaction Submitted    │                 │
     │          └──────────────────────────┘                 │
     │                 │                                      │
     │                 ▼                                      │
     │          ┌──────────┐                                 │
     │          │ Confirmed│                                 │
     │          │ Event    │                                 │
     │          └──────────┘                                 │
     │                 │                                      │
     │                 ▼                                      │
     │          ┌──────────┐                                 │
     │          │ Payment  │                                 │
     │          │ Verified │                                 │
     │          └──────────┘                                 │
     │                 │                                      │
     └─────────────────┴──────────────────────────────────────┘
```

## Security Architecture

### Threat Model

**Threats Identified**:
1. **Replay Attacks**: Reusing valid transaction signatures
2. **Man-in-the-Middle**: Intercepting and modifying transactions
3. **DDoS Attacks**: Overwhelming the system with requests
4. **Insufficient Funds**: Attempting payments without sufficient balance
5. **Transaction Manipulation**: Modifying transaction parameters

### Security Controls

1. **Cryptographic Nonces**: Prevent replay attacks
2. **TLS 1.3**: Encrypt all communications
3. **Rate Limiting**: Token bucket algorithm
4. **Input Validation**: Strict schema validation
5. **On-Chain Verification**: All payments verified on blockchain

### Security Layers

```
┌─────────────────────────────────────────┐
│     Application Security Layer         │
│  - Input Validation                    │
│  - Authentication/Authorization        │
│  - Rate Limiting                       │
└─────────────────────────────────────────┘
┌─────────────────────────────────────────┐
│     Transport Security Layer            │
│  - TLS 1.3 Encryption                   │
│  - Certificate Pinning                 │
│  - HSTS                                 │
└─────────────────────────────────────────┘
┌─────────────────────────────────────────┐
│     Blockchain Security Layer           │
│  - Cryptographic Signatures              │
│  - On-Chain Verification                │
│  - Consensus Mechanism                  │
└─────────────────────────────────────────┘
```

## Scalability Architecture

### Horizontal Scaling

- **Stateless Services**: All services are stateless, enabling horizontal scaling
- **Load Balancing**: Round-robin or least-connections algorithm
- **Auto-Scaling**: Kubernetes HPA or AWS Auto Scaling Groups
- **Database Sharding**: Partition data by payment ID or timestamp

### Caching Strategy

- **L1 Cache**: In-memory cache (Redis) for hot data
- **L2 Cache**: Distributed cache (Memcached) for warm data
- **CDN**: Static assets and API responses
- **Cache Invalidation**: TTL-based or event-driven

### Performance Optimization

- **Connection Pooling**: Reuse RPC connections
- **Request Batching**: Batch multiple operations
- **Async Processing**: Non-blocking I/O operations
- **Database Indexing**: Optimize query performance

## Deployment Architecture

### Containerization

- **Docker**: Container images for all services
- **Kubernetes**: Orchestration and management
- **Helm Charts**: Package management
- **Service Mesh**: Istio or Linkerd for traffic management

### Infrastructure as Code

- **Terraform**: Infrastructure provisioning
- **Ansible**: Configuration management
- **CI/CD**: GitHub Actions or GitLab CI
- **Monitoring**: Prometheus + Grafana

### Multi-Region Deployment

- **Primary Region**: US-East-1 (Primary)
- **Secondary Region**: EU-West-1 (Failover)
- **Global Load Balancer**: Route traffic to nearest region
- **Data Replication**: Async replication between regions

---

**Last Updated**: January 2025 | **Version**: 1.0.0

