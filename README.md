# Distributed Payment Gateway

A fault-tolerant distributed payment system implementing Stripe-like functionality with secure multi-bank transaction coordination using gRPC, Consul service discovery, and two-phase commit protocol.


## Overview

This project implements a miniature version of Stripe - a distributed payment gateway that interfaces with clients and multiple bank servers to manage secure transactions. The system provides robust authentication, atomic cross-bank transfers, and failure handling mechanisms including offline payment queuing and automatic rollback.

## Features

- **Secure Authentication & Authorization**: JWT-based token authentication with 1-hour validity
- **Two-Phase Commit (2PC) Protocol**: Ensures atomic transactions across multiple banks with configurable timeout
- **Service Discovery**: Dynamic bank registration and discovery using Consul with health checks
- **SSL/TLS Encryption**: End-to-end encrypted communication between all components
- **Transaction Idempotency**: UUID-based duplicate detection preventing double processing
- **Offline Payment Queuing**: Automatic retry mechanism with 0.2s intervals during gateway unavailability
- **Comprehensive Logging**: Interceptor-based request/response logging for debugging and monitoring
- **Fault Tolerance**: Automatic rollback on failures with state persistence

## System Architecture

The system follows a three-tier architecture:

### Components

1. **Client (`client/client.py`)**
   - Handles user interactions (login, balance checks, transfers, transaction history)
   - Assigns unique transaction IDs (UUIDs)
   - Queues transactions during offline scenarios using background thread
   - Implements automatic retry mechanism

2. **Gateway (`server/gateway.py`)**
   - Central coordinator between clients and banks
   - Manages user authentication and JWT token issuance
   - Orchestrates cross-bank transactions using 2PC protocol
   - Enforces authorization via JWT interceptor
   - Maintains transaction registry for idempotency

3. **Bank Servers (`server/bank.py`)**
   - Represents individual banks with unique names
   - Handles user authentication and password verification
   - Manages account balances and transaction history
   - Participates in 2PC protocol (prepare, commit, abort phases)
   - Persists user data locally

### Communication

- All interactions use **gRPC** with SSL/TLS secure channels
- Certificates generated for mutual authentication
- Public keys stored in Consul for trusted distribution

## Project Structure

```
Distributed Payment Gateway/
├── protofiles/
│   ├── service.proto           # gRPC service definitions
│   ├── service_pb2.py          # Generated protobuf messages
│   └── service_pb2_grpc.py     # Generated gRPC stubs
├── client/
│   └── client.py               # Client implementation
├── server/
│   ├── gateway.py              # Gateway coordinator
│   ├── bank.py                 # Bank server implementation
│   └── user.json               # User data and balances
├── config.json                 # Configuration (2PC timeout, etc.)
└── README.md                   # This file
```

## Prerequisites

- Python 3.7+
- Consul (running locally or accessible)
- Required Python packages:
  ```bash
  pip install grpcio grpcio-tools consul pyjwt cryptography
  ```

## Installation

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd Distributed-Payment-Gateway
   ```

2. **Generate gRPC code** (if not already generated)
   ```bash
   python -m grpc_tools.protoc -I./protofiles --python_out=./protofiles --grpc_python_out=./protofiles ./protofiles/service.proto
   ```

3. **Start Consul**
   ```bash
   consul agent -dev
   ```

4. **Configure settings** (optional)
   - Edit `config.json` to adjust 2PC timeout and other parameters

## Running the System

### 1. Start the Gateway
```bash
python server/gateway.py
```
The gateway will register itself with Consul and start listening for client requests.

### 2. Start Bank Server(s)
Run one or more bank instances with unique names:
```bash
python server/bank.py --name BankA
python server/bank.py --name BankB
```
Each bank registers with Consul and begins serving requests.

### 3. Run the Client
Launch the client with optional credentials:
```bash
python client/client.py [<bank_name> <username> <password>]
```

**Example:**
```bash
python client/client.py BankA alice password123
```

### Client Operations

Once logged in, the client supports:
- **View Balance**: Check current account balance
- **Transfer Money**: Initiate cross-bank or intra-bank transfers
- **Transaction History**: View past transactions
- **Logout**: End session

## Key Design Highlights

### Two-Phase Commit Protocol

The gateway coordinates atomic transactions:

1. **Prepare Phase**: 
   - Sends `PrepareDebit` to source bank
   - Sends `PrepareCredit` to destination bank
   - Both must succeed within configurable timeout

2. **Commit/Abort Phase**:
   - If both prepare: Send `CommitDebit` and `CommitCredit`
   - If any fails: Send `AbortDebit` and `AbortCredit` to rollback

### Transaction Idempotency

- **Client**: Assigns unique UUID to each transaction
- **Gateway**: Maintains `transaction_ids` set, rejects duplicates
- **Banks**: Track IDs in `prepare_transaction`, prevent double processing
- **Persistence**: Completed transaction IDs saved to `gateway_data.json`

**Correctness Guarantee**: Each transaction affects balances exactly once, even with retries.

### Offline Payment Handling

When the gateway is unreachable:
1. Client queues transaction in `transaction_queue`
2. Background thread (`offline_txn_worker`) checks gateway availability every 0.2s
3. Automatically resends queued transactions when gateway recovers
4. Results stored in `queue_output` for user notification

### Authentication Flow

1. Client sends credentials to gateway (`Login`)
2. Gateway forwards to appropriate bank (`Authenticate`)
3. Bank verifies password hash from `user.json`
4. Gateway issues JWT token (1-hour validity)
5. Token validated via `JwtInterceptor` for protected endpoints

### Fault Tolerance

- **State Persistence**: Gateway and banks save state to JSON files on shutdown
- **Crash Handling**: Ongoing transactions abort on crash; clients retry
- **Recovery**: Completed transactions preserved post-restart
- **Automatic Rollback**: 2PC ensures no partial commits

## Configuration

Edit `config.json` to customize:
```json
{
  "TIMEOUT2PC": 5.0,
  "JWT_SECRET": "your-secret-key",
  "TOKEN_EXPIRY_HOURS": 1
}
```

## Logging

- **Gateway**: `gateway_interceptor.log`
- **Banks**: `<bank_name>_interceptor.log`

Logs capture method names, request/response data, and errors for debugging.

## Assumptions

- Usernames are unique within each bank (one account per user)
- Consul is always available and serves as trusted authority
- Server crashes abort ongoing transactions (clients must retry)
- Expired tokens require re-authentication for offline transactions

## Security Features

- **SSL/TLS**: All gRPC communication encrypted
- **JWT Tokens**: Stateless authentication with signature verification
- **Password Hashing**: Stored credentials hashed in `user.json`
- **Certificate-Based Auth**: Mutual authentication between components
- **Token Registry**: Prevents unauthorized access to protected endpoints


## Troubleshooting

**Issue**: Client cannot connect to gateway
- Ensure Consul is running
- Verify gateway has registered with Consul
- Check SSL certificates are properly generated

**Issue**: Transaction fails with timeout
- Increase `TIMEOUT2PC` in `config.json`
- Check bank server logs for errors
- Verify network connectivity between components

**Issue**: Token expired during offline payments
- Re-authenticate before retrying queued transactions
- Increase `TOKEN_EXPIRY_HOURS` if needed
