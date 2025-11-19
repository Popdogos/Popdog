# Popdog Protocol Security Architecture

## Document Information

- **Version**: 1.0.0
- **Classification**: Public Security Documentation
- **Last Updated**: January 2025
- **Status**: Production Ready

## Table of Contents

1. [Security Model](#security-model)
2. [Threat Model](#threat-model)
3. [Cryptographic Security](#cryptographic-security)
4. [Network Security](#network-security)
5. [Application Security](#application-security)
6. [Infrastructure Security](#infrastructure-security)
7. [Compliance & Auditing](#compliance--auditing)

## Security Model

### Zero-Trust Architecture

Popdog implements a **zero-trust security model** where:

- **No Implicit Trust**: Every request is verified regardless of source
- **Least Privilege**: Services operate with minimum required permissions
- **Defense in Depth**: Multiple security layers protect critical assets
- **Continuous Verification**: Ongoing validation of identities and permissions

### Security Principles

1. **Confidentiality**: Protect sensitive data at rest and in transit
2. **Integrity**: Ensure data and transactions cannot be tampered with
3. **Availability**: Maintain service availability under attack
4. **Non-Repudiation**: Provide cryptographic proof of transactions
5. **Authentication**: Verify identities of all parties
6. **Authorization**: Enforce access controls

## Threat Model

### Threat Categories

#### 1. Cryptographic Attacks

**Threats**:
- Private key compromise
- Signature forgery
- Hash collision attacks
- Replay attacks

**Mitigations**:
- Hardware Security Modules (HSM) for key storage
- Ed25519 signatures (quantum-resistant)
- SHA-256 hashing (collision-resistant)
- Cryptographic nonces for replay prevention

#### 2. Network Attacks

**Threats**:
- Man-in-the-Middle (MITM) attacks
- Distributed Denial of Service (DDoS)
- DNS spoofing
- SSL/TLS downgrade attacks

**Mitigations**:
- TLS 1.3 with certificate pinning
- Rate limiting and DDoS protection
- DNSSEC for DNS security
- HSTS (HTTP Strict Transport Security)

#### 3. Application Attacks

**Threats**:
- SQL injection
- Cross-Site Scripting (XSS)
- Cross-Site Request Forgery (CSRF)
- Input validation bypass

**Mitigations**:
- Parameterized queries
- Content Security Policy (CSP)
- CSRF tokens
- Strict input validation

#### 4. Infrastructure Attacks

**Threats**:
- Unauthorized access to servers
- Container escape
- Privilege escalation
- Supply chain attacks

**Mitigations**:
- Network segmentation
- Container security scanning
- Least privilege access
- Dependency vulnerability scanning

## Cryptographic Security

### Key Management

#### Key Generation

```typescript
import crypto from 'crypto';
import { Keypair } from '@solana/web3.js';

class KeyManagementService {
    /**
     * Generates cryptographically secure keypair
     * 
     * Uses CSPRNG (Cryptographically Secure Pseudorandom Number Generator)
     * 
     * @returns Keypair
     */
    generateKeypair(): Keypair {
        // Generate 32 random bytes (256 bits) for private key
        const privateKey = crypto.randomBytes(32);
        
        // Generate Ed25519 keypair from seed
        return Keypair.fromSeed(privateKey);
    }
    
    /**
     * Derives keypair from seed phrase (BIP39)
     * 
     * @param mnemonic - BIP39 mnemonic phrase
     * @param derivationPath - BIP44 derivation path
     * @returns Keypair
     */
    deriveKeypair(mnemonic: string, derivationPath: string): Keypair {
        // Implement BIP39/BIP44 derivation
        // This is a simplified example
        const seed = this.mnemonicToSeed(mnemonic);
        const derivedKey = this.deriveFromPath(seed, derivationPath);
        return Keypair.fromSeed(derivedKey);
    }
}
```

#### Key Storage

**Hardware Security Modules (HSM)**:
- Private keys never leave HSM
- All cryptographic operations performed in HSM
- Tamper-resistant hardware
- FIPS 140-2 Level 3 certified

**Software Key Storage**:
- Encrypted at rest (AES-256-GCM)
- Encrypted in transit (TLS 1.3)
- Key rotation policies
- Access logging and auditing

### Signature Security

#### Ed25519 Signature Scheme

**Properties**:
- **Curve**: Ed25519 (Edwards-curve Digital Signature Algorithm)
- **Key Size**: 256 bits (32 bytes)
- **Signature Size**: 512 bits (64 bytes)
- **Security Level**: 128 bits
- **Quantum Resistance**: Post-quantum secure

**Implementation**:

```typescript
import nacl from 'tweetnacl';

class SignatureService {
    /**
     * Signs message with Ed25519
     * 
     * @param message - Message to sign
     * @param keypair - Ed25519 keypair
     * @returns Signature (64 bytes)
     */
    sign(message: Uint8Array, keypair: Keypair): Uint8Array {
        return nacl.sign.detached(message, keypair.secretKey);
    }
    
    /**
     * Verifies Ed25519 signature
     * 
     * @param message - Original message
     * @param signature - Signature to verify
     * @param publicKey - Public key
     * @returns boolean
     */
    verify(
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
}
```

### Nonce Management

**Purpose**: Prevent replay attacks

**Implementation**:

```typescript
class NonceManager {
    private cache: Cache;
    private nonceLength: number = 32; // 256 bits
    
    /**
     * Generates cryptographically secure nonce
     * 
     * @returns Base64-encoded nonce
     */
    generateNonce(): string {
        const randomBytes = crypto.randomBytes(this.nonceLength);
        return randomBytes.toString('base64');
    }
    
    /**
     * Validates and marks nonce as used
     * 
     * @param nonce - Nonce to validate
     * @param ttl - Time-to-live in seconds
     * @returns boolean - True if nonce is valid and unused
     */
    async validateAndUse(nonce: string, ttl: number = 3600): Promise<boolean> {
        const key = `nonce:${nonce}`;
        
        // Check if nonce already used
        const exists = await this.cache.exists(key);
        if (exists) {
            return false; // Replay attack detected
        }
        
        // Mark as used
        await this.cache.set(key, 'used', ttl);
        return true;
    }
}
```

## Network Security

### Transport Layer Security (TLS)

**Configuration**:
- **Protocol**: TLS 1.3 only
- **Cipher Suites**: 
  - TLS_AES_256_GCM_SHA384
  - TLS_CHACHA20_POLY1305_SHA256
  - TLS_AES_128_GCM_SHA256
- **Certificate**: ECDSA P-384 or RSA 4096-bit
- **Certificate Pinning**: Enabled for mobile clients
- **Perfect Forward Secrecy**: Enabled

### API Security

#### Authentication

**JWT (JSON Web Tokens)**:

```typescript
interface JWTPayload {
    sub: string;        // Subject (user ID)
    iss: string;       // Issuer
    aud: string;       // Audience
    exp: number;       // Expiration time
    iat: number;       // Issued at
    jti: string;       // JWT ID (nonce)
    roles: string[];   // User roles
}

class AuthenticationService {
    /**
     * Generates JWT token
     * 
     * @param payload - JWT payload
     * @returns Signed JWT token
     */
    generateToken(payload: JWTPayload): string {
        return jwt.sign(payload, this.privateKey, {
            algorithm: 'ES256', // ECDSA with P-256 and SHA-256
            expiresIn: '1h',
            issuer: 'popdog-protocol',
        });
    }
    
    /**
     * Verifies JWT token
     * 
     * @param token - JWT token to verify
     * @returns Decoded payload
     */
    verifyToken(token: string): JWTPayload {
        return jwt.verify(token, this.publicKey, {
            algorithms: ['ES256'],
            issuer: 'popdog-protocol',
        }) as JWTPayload;
    }
}
```

#### Rate Limiting

**Token Bucket Algorithm**:

```typescript
class RateLimiter {
    private buckets: Map<string, TokenBucket> = new Map();
    
    /**
     * Checks if request is allowed
     * 
     * @param key - Rate limit key (IP, user ID, etc.)
     * @param limit - Request limit
     * @param window - Time window in seconds
     * @returns boolean
     */
    isAllowed(
        key: string,
        limit: number = 100,
        window: number = 60
    ): boolean {
        let bucket = this.buckets.get(key);
        
        if (!bucket) {
            bucket = new TokenBucket(limit, limit / window);
            this.buckets.set(key, bucket);
        }
        
        return bucket.consume(1);
    }
}
```

## Application Security

### Input Validation

**Schema Validation**:

```typescript
import { z } from 'zod';

const PaymentRequestSchema = z.object({
    amount: z.bigint().positive(),
    token: z.string().regex(/^[1-9A-HJ-NP-Za-km-z]{32,44}$/), // Base58 Solana address
    recipient: z.string().regex(/^[1-9A-HJ-NP-Za-km-z]{32,44}$/),
    nonce: z.string().length(44), // Base64 nonce
    metadata: z.record(z.any()).optional(),
});

class ValidationService {
    validatePaymentRequest(data: unknown): PaymentRequest {
        return PaymentRequestSchema.parse(data);
    }
}
```

### SQL Injection Prevention

**Parameterized Queries**:

```typescript
class DatabaseService {
    /**
     * Executes parameterized query
     * 
     * @param query - SQL query with placeholders
     * @param params - Query parameters
     * @returns Query result
     */
    async executeQuery<T>(
        query: string,
        params: any[]
    ): Promise<T[]> {
        // Use parameterized queries to prevent SQL injection
        return this.db.query(query, params);
    }
}
```

### XSS Prevention

**Content Security Policy (CSP)**:

```
Content-Security-Policy: 
    default-src 'self';
    script-src 'self' 'unsafe-inline';
    style-src 'self' 'unsafe-inline';
    img-src 'self' data: https:;
    connect-src 'self' https://api.mainnet-beta.solana.com;
    font-src 'self';
    frame-ancestors 'none';
```

## Infrastructure Security

### Container Security

**Dockerfile Best Practices**:

```dockerfile
# Use minimal base image
FROM node:18-alpine AS builder

# Run as non-root user
RUN addgroup -g 1001 -S appuser && \
    adduser -S appuser -u 1001

# Copy files
COPY --chown=appuser:appuser . /app
WORKDIR /app

# Install dependencies
RUN npm ci --only=production

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
    CMD node healthcheck.js

# Start application
CMD ["node", "dist/index.js"]
```

### Network Segmentation

**Security Zones**:
- **Public Zone**: Internet-facing services (API Gateway)
- **DMZ**: Application services
- **Private Zone**: Database and internal services
- **Management Zone**: Monitoring and administration

### Secrets Management

**Environment Variables**:
- Never commit secrets to repository
- Use secret management service (AWS Secrets Manager, HashiCorp Vault)
- Rotate secrets regularly
- Audit secret access

## Compliance & Auditing

### Security Auditing

**Audit Logs**:
- All authentication attempts
- All authorization decisions
- All payment transactions
- All configuration changes
- All security events

**Log Format**:

```typescript
interface AuditLog {
    timestamp: string;        // ISO 8601
    event: string;           // Event type
    userId?: string;         // User ID (if applicable)
    ipAddress: string;       // Client IP
    userAgent: string;       // Client user agent
    resource: string;        // Resource accessed
    action: string;          // Action performed
    result: 'success' | 'failure';
    details?: Record<string, any>;
}
```

### Compliance Standards

- **SOC 2 Type II**: Security and availability controls
- **PCI DSS**: Payment card industry security (if applicable)
- **GDPR**: General Data Protection Regulation
- **CCPA**: California Consumer Privacy Act

---

**Last Updated**: January 2025 | **Version**: 1.0.0 | **Status**: Production Ready

