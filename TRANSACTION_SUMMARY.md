# Transaction System - Complete Reference Guide

## Overview
The ticket platform has a comprehensive transaction system for managing ticket purchases and sales between buyers and sellers. Transactions track the entire lifecycle from reservation through confirmation/cancellation.

---

## 1. Core Entities (DBModel)

### Transaction (Main Entity)
**File**: `TicketPlatFormServer/DBModel/Transaction.cs`

**Purpose**: Central entity representing a single transaction between buyer and seller

**Key Properties**:
| Property | Type | Purpose |
|----------|------|----------|
| `Id` | `long` | Primary key |
| `BuyerId` | `long` | FK to buyer user |
| `SellerId` | `long` | FK to seller user |
| `StatusId` | `long` | FK to TransactionStatus |
| `Amount` | `int?` | Total transaction amount (sum of TransactionItem.TotalPrice) |
| `ReservedAt` | `DateTime?` | Reservation timestamp |
| `ReservationExpiresAt` | `DateTime?` | Reservation expiration time |
| `ConfirmedAt` | `DateTime?` | Purchase confirmation timestamp |
| `AutoConfirmAt` | `DateTime?` | Scheduled auto-confirmation time |
| `CancelledAt` | `DateTime?` | Cancellation timestamp |
| `ConfirmedById` | `long?` | FK to who confirmed (USER or SYSTEM) |
| `CreatedAt` | `DateTime?` | Creation timestamp |
| `DeletedAt` | `DateTime?` | Soft delete timestamp |

**Relationships**:
- `TransactionItems` (1:N) - Individual tickets in the transaction
- `TransactionStatus` (N:1) - Current status reference
- `Payments` (1:N) - Payment records
- `Refunds` (1:N) - Refund records
- `Disputes` (1:N) - Active disputes
- `Escrow` (1:1) - Escrow holding funds
- `ChatRooms` (1:N) - Related chat conversations
- `TicketVerifications` (1:N) - Ticket verification records

---

### TransactionItem (Line Items)
**File**: `TicketPlatFormServer/DBModel/TransactionItem.cs`

**Purpose**: Individual ticket purchase record (one transaction can have multiple items)

**Key Properties**:
| Property | Type | Purpose |
|----------|------|----------|
| `Id` | `long` | Primary key |
| `TransactionId` | `long` | FK to Transaction |
| `TicketId` | `long` | FK to Ticket being purchased |
| `Quantity` | `int` | Number of tickets (usually 1) |
| `UnitPrice` | `int` | Price per ticket |
| `TotalPrice` | `int` | UnitPrice × Quantity |
| `CreatedAt` | `DateTime?` | Creation timestamp |

---

### TransactionStatus (State Machine)
**File**: `TicketPlatFormServer/DBModel/TransactionStatus.cs`

**Purpose**: Lookup table for transaction states

**Valid Status Codes**:
- `reserved` - Initial state, awaiting payment
- `pending_payment` - Payment in progress
- `paid` - Payment confirmed, awaiting buyer confirmation
- `confirmed` - Buyer confirmed receipt
- `completed` - Transaction fully complete
- `cancelled` - Transaction cancelled
- `refunded` - Refund issued

**Properties**:
| Property | Type | Purpose |
|----------|------|----------|
| `Id` | `long` | Primary key |
| `Code` | `string` | Status identifier (e.g., "reserved", "paid") |
| `NameKo` | `string?` | Korean display name |
| `IsActive` | `bool?` | Whether status is currently usable |
| `SortOrder` | `int` | Display order |

---

### PaymentTransaction (Payment Events)
**File**: `TicketPlatFormServer/DBModel/PaymentTransaction.cs`

**Purpose**: Audit trail of all payment gateway transactions (approvals, cancellations, etc.)

**Key Properties**:
| Property | Type | Purpose |
|----------|------|----------|
| `Id` | `ulong` | Primary key |
| `PaymentId` | `ulong` | FK to Payment |
| `TransactionKey` | `string` | Toss Payments gateway transaction key |
| `TransactionType` | `string` | PAYMENT, CANCEL, PARTIAL_CANCEL |
| `Amount` | `ulong` | Transaction amount |
| `BalanceAmount` | `ulong?` | Remaining balance after partial cancel |
| `TaxFreeAmount` | `ulong` | Tax-free portion |
| `Currency` | `string` | ISO-4217 currency code |
| `Status` | `string` | DONE, FAILED, PENDING |
| `Reason` | `string?` | Cancellation reason |
| `TossResponse` | `string?` | Encrypted full API response |
| `EventAt` | `DateTime?` | Event timestamp from payment gateway |
| `CreatedAt` | `DateTime?` | DB storage timestamp |

---

## 2. Repository Layer

### ITransactionRepository & TransactionRepository
**Files**: 
- Interface: `Repository/Transaction/ITransactionRepository.cs`
- Implementation: `Repository/Transaction/TransactionRepository.cs`

**Uses**: EF Core + Dapper (hybrid approach)

**Key Methods**:

#### Read Operations
```csharp
// Get transaction by ID
Task<Transaction?> GetTransactionById(long transactionId)
// Query: includes Status relationship

// Get transaction with details (items, status)
Task<Transaction?> GetTransactionWithDetailsAsync(long transactionId)
// Query: includes Status + TransactionItems with AsSplitQuery()

// Find expired pending transactions (for cleanup)
Task<List<Transaction>> GetExpiredPendingTransactionsAsync(DateTime utcNow)
// Filters: pending_payment status + ReservationExpiresAt < now

// Find transactions due for auto-confirmation
Task<List<Transaction>> GetAutoConfirmDueTransactionsAsync(DateTime utcNow)
// Filters: paid status + AutoConfirmAt <= now + no manual confirmation yet

// Validate transaction ownership
Task<bool> ValidateTransactionOwnership(long transactionId, long buyerId, long sellerId)
// Uses Dapper for simple count query

// Get payment preview data
Task<PaymentPreviewReadModel?> GetPaymentPreviewAsync(long transactionId, int buyerId)
// Complex JOIN: transactions → items → tickets → events
// Returns: amount, seat info, event details for UI
```

#### Write Operations
```csharp
// Create new transaction
Task<Transaction> CreateTransactionAsync(Transaction transaction)
// Sets CreatedAt = UtcNow, calls SaveChangesAsync()

// Update transaction status
Task UpdateTransactionStatusAsync(long transactionId, long statusId)
// Uses Dapper with transaction support

// Update cancellation timestamp
Task UpdateTransactionCancelledAtAsync(long transactionId, DateTime cancelledAt)
// Uses Dapper with transaction support
```

**Important**: Methods use Dapper when within an active EF transaction:
```csharp
if (context.Database.CurrentTransaction != null)
{
    // Share EF transaction with Dapper
    await connection.ExecuteAsync(query, ..., transaction: currentTransaction.GetDbTransaction());
}
```

---

### ITransactionItemRepository & TransactionItemRepository
**Files**:
- Interface: `Repository/Transaction/ITransactionItemRepository.cs`
- Implementation: `Repository/Transaction/TransactionItemRepository.cs`

**Key Methods**:
```csharp
// Create transaction line item
Task<TransactionItem> CreateTransactionItemAsync(TransactionItem item)
// Sets CreatedAt = UtcNow, calls SaveChangesAsync()
```

---

## 3. Service Layer

### ITransactionService & TransactionService
**Files**:
- Interface: `Services/Transaction/ITransactionService.cs`
- Implementation: `Services/Transaction/TransactionService.cs`

**Dependencies**:
- `ITransactionHistoryRepository` - History/audit queries
- `ILogger<TransactionService>` - Structured logging

**Key Methods**:

#### Purchase History
```csharp
Task<TransactionHistoryRespDto> GetPurchaseHistoryAsync(
    int userId,
    string? status,          // "reserved", "paid", etc. (comma-separated)
    string? period,          // "1w", "1m", "3m", "6m", "all"
    string? sortBy,          // "latest" or "oldest"
    string? cursor,          // Base64-encoded pagination
    int? limit)              // Default 20, max 50
```

#### Sales History
```csharp
Task<TransactionHistoryRespDto> GetSalesHistoryAsync(
    int userId,
    string? status,
    string? period,
    string? sortBy,
    string? cursor,
    int? limit)
```

**Features**:
- **Parameter Validation**: Status and period enum validation
- **Cursor-based Pagination**: Base64-encoded `{Id, CreatedAt}` for stateless pagination
- **Performance**: Only fetches total count on first page
- **Logging**: Comprehensive operation logging

**Response Format**:
```csharp
public class TransactionHistoryRespDto
{
    public List<TransactionHistoryItem> Items { get; set; }
    public string? NextCursor { get; set; }
    public bool HasMore { get; set; }
    public long? TotalCount { get; set; }  // Only on first page
}
```

---

## 4. Controllers

### TransactionController
**File**: `Controllers/TransactionController.cs`

**Routes**:
```
GET /api/transactions/purchases  - Get purchase history
GET /api/transactions/sales      - Get sales history
```

**Authentication**: `[Authorize]` - Requires JWT token

**Query Parameters**:
| Param | Type | Default | Example |
|-------|------|---------|----------|
| `status` | string | null | "paid,confirmed" |
| `period` | string | "all" | "1m" |
| `sortBy` | string | "latest" | "oldest" |
| `cursor` | string | null | Base64 pagination token |
| `limit` | int | 20 | 50 |

---

## 5. Background Services

### TransactionAutoConfirmService
**File**: `Services/BackgroundServices/TransactionAutoConfirmService.cs`

**Purpose**: Automatically confirm transactions and release escrow after payment

**Execution**:
- **Interval**: Every 10 minutes
- **Trigger**: `AutoConfirmAt <= DateTime.UtcNow`

**Process**:
1. Query transactions due for auto-confirmation (status = "paid")
2. Check for active disputes (PENDING, IN_REVIEW)
3. If no disputes: call `PaymentService.ReleaseEscrowAsync()`
4. Send notification to buyer: REVIEW_REQUEST
5. Log completion/failures

**Key Dependencies**:
- `ITransactionRepository` - Get due transactions
- `IDisputeRepository` - Check for active disputes
- `IPaymentService` - Release escrow
- `INotificationService` - Send review request
- `TicketContext` - Get seller nickname

---

### TransactionReservationCleanupService
**File**: `Services/BackgroundServices/TransactionReservationCleanupService.cs`

**Purpose**: Cancel expired reservations (cleanup)

**Execution**:
- **Interval**: Every 5 minutes
- **Trigger**: `ReservationExpiresAt < DateTime.UtcNow` AND status = "pending_payment"

**Process**:
1. Query expired reservations
2. Update status to "cancelled"
3. Update `CancelledAt` timestamp
4. Release held tickets back to inventory

---

## 6. Transaction Lifecycle

### State Flow
```
RESERVED
    ↓ (payment initiated)
PENDING_PAYMENT
    ↓ (payment confirmed)
PAID
    ↓ (buyer confirms OR auto-confirm after delay)
CONFIRMED
    ↓ (manual close or refund finalized)
COMPLETED

Or at any stage:
CANCELLED
    ↓ (refund issued)
REFUNDED
```

### Key Timestamps
| Timestamp | Set When | Purpose |
|-----------|----------|----------|
| `ReservedAt` | Transaction created | Track reservation start |
| `ReservationExpiresAt` | On creation | Deadline for payment (usually 15min) |
| `ConfirmedAt` | Buyer confirms receipt | Track purchase completion |
| `AutoConfirmAt` | Payment confirmed | Scheduled auto-confirm (usually 7+ days) |
| `CancelledAt` | Cancellation triggered | Track cancellation event |
| `DeletedAt` | Soft delete | Logical deletion |

---

## 7. Critical Business Rules

### Payment & Escrow
- ⚠️ Funds are held in **escrow** during the transaction lifecycle
- **Escrow Release**: Triggered by `ReleaseEscrowAsync()` (auto-confirm or manual)
- **Refund Processing**: Handled by `PaymentService`

### Transaction Ownership
- Only buyer and seller can view/modify their transactions
- **Validation**: `ValidateTransactionOwnership(txnId, buyerId, sellerId)`

### Status Transitions
- `reserved` → `pending_payment` (payment initiated)
- `pending_payment` → `paid` (payment gateway confirms)
- `paid` → `confirmed` (buyer confirms OR timeout)
- `*` → `cancelled` (at any stage)

### Dispute Handling
- Active disputes **block auto-confirmation**
- Disputes checked in: `TransactionAutoConfirmService`
- Statuses: PENDING, IN_REVIEW

### Concurrent Modifications
- ⚠️ Use optimistic locking or transaction isolation for concurrent updates
- Dapper calls within EF transaction use `GetDbTransaction()` for safety

---

## 8. Key Files Reference

| Component | File Path |
|-----------|----------|
| **Transaction Entity** | `DBModel/Transaction.cs` |
| **TransactionItem Entity** | `DBModel/TransactionItem.cs` |
| **TransactionStatus Enum** | `DBModel/TransactionStatus.cs` |
| **PaymentTransaction Entity** | `DBModel/PaymentTransaction.cs` |
| **Transaction Repository Interface** | `Repository/Transaction/ITransactionRepository.cs` |
| **Transaction Repository** | `Repository/Transaction/TransactionRepository.cs` |
| **TransactionItem Repository** | `Repository/Transaction/TransactionItemRepository.cs` |
| **TransactionHistory Repository** | `Repository/Transaction/TransactionHistoryRepository.cs` |
| **Transaction Service Interface** | `Services/Transaction/ITransactionService.cs` |
| **Transaction Service** | `Services/Transaction/TransactionService.cs` |
| **Transaction Controller** | `Controllers/TransactionController.cs` |
| **Auto-Confirm Service** | `Services/BackgroundServices/TransactionAutoConfirmService.cs` |
| **Cleanup Service** | `Services/BackgroundServices/TransactionReservationCleanupService.cs` |

---

## 9. Common Operations

### Creating a Transaction
```csharp
// 1. Create Transaction entity
var transaction = new Transaction
{
    BuyerId = buyerId,
    SellerId = sellerId,
    StatusId = pendingPaymentStatusId,  // "reserved" status
    ReservedAt = DateTime.UtcNow,
    ReservationExpiresAt = DateTime.UtcNow.AddMinutes(15),
    Amount = totalAmount
};

// 2. Create via repository
var created = await transactionRepository.CreateTransactionAsync(transaction);

// 3. Create transaction items
foreach (var ticket in tickets)
{
    var item = new TransactionItem
    {
        TransactionId = created.Id,
        TicketId = ticket.Id,
        Quantity = quantity,
        UnitPrice = ticket.Price,
        TotalPrice = ticket.Price * quantity
    };
    await itemRepository.CreateTransactionItemAsync(item);
}
```

### Updating Transaction Status
```csharp
// Get status entity
var paidStatus = await transactionRepository.GetTransactionStatusByCodeAsync("paid");

// Update status
await transactionRepository.UpdateTransactionStatusAsync(transactionId, paidStatus.Id);
```

### Querying Purchase History
```csharp
var history = await transactionService.GetPurchaseHistoryAsync(
    userId: 123,
    status: "paid,confirmed",    // comma-separated
    period: "3m",                // last 3 months
    sortBy: "latest",
    cursor: null,                // first page
    limit: 20
);

// Result: { Items, NextCursor, HasMore, TotalCount }
```

### Checking for Auto-Confirm
```csharp
var dueTransactions = await transactionRepository
    .GetAutoConfirmDueTransactionsAsync(DateTime.UtcNow);

// Background service processes these every 10 minutes
```

---

## 10. Database Indexes (Recommended)

```sql
-- Core lookups
CREATE INDEX idx_transaction_buyer_id ON transactions(buyer_id);
CREATE INDEX idx_transaction_seller_id ON transactions(seller_id);
CREATE INDEX idx_transaction_status_id ON transactions(status_id);
CREATE INDEX idx_transaction_created_at ON transactions(created_at DESC);

-- For auto-confirm queries
CREATE INDEX idx_transaction_auto_confirm ON transactions(auto_confirm_at)
WHERE auto_confirm_at IS NOT NULL AND cancelled_at IS NULL;

-- For cleanup queries
CREATE INDEX idx_transaction_reservation_expires ON transactions(reservation_expires_at)
WHERE reservation_expires_at IS NOT NULL AND cancelled_at IS NULL;

-- For soft deletes
CREATE INDEX idx_transaction_deleted_at ON transactions(deleted_at);

-- TransactionItems
CREATE INDEX idx_transaction_item_transaction_id ON transaction_items(transaction_id);
CREATE INDEX idx_transaction_item_ticket_id ON transaction_items(ticket_id);
```

---

## 11. Error Handling Patterns

All operations follow the **AppException** pattern:

```csharp
// Business rule violation
if (transaction == null)
{
    throw new AppException("거래를 찾을 수 없습니다.", HttpStatusCode.NotFound);
}

// Database error with context
try
{
    await context.SaveChangesAsync();
}
catch (DbUpdateException ex)
{
    throw new AppException(
        "거래 저장 실패",
        HttpStatusCode.InternalServerError,
        ex);  // Inner exception for logging
}
```

---

## 12. Testing Considerations

### Unit Tests (Service Layer)
```csharp
[Fact]
public async Task GetPurchaseHistory_WithValidParams_ReturnsResult()
{
    // Mock ITransactionHistoryRepository
    // Test parameter validation
    // Test cursor parsing/creation
    // Test pagination logic
}
```

### Integration Tests (Repository Layer)
```csharp
[Fact]
public async Task CreateTransaction_InsertsAndReturnedWithId()
{
    // Use actual database or Testcontainers
    // Test SaveChangesAsync() execution
    // Verify CreatedAt is set
    // Verify relationships are loaded
}
```

---

## Quick Reference

**Creating**: `CreateTransactionAsync()`, `CreateTransactionItemAsync()`

**Reading**: `GetTransactionById()`, `GetTransactionWithDetailsAsync()`

**Updating**: `UpdateTransactionStatusAsync()`, `UpdateTransactionCancelledAtAsync()`

**Querying**: `GetPurchaseHistoryAsync()`, `GetSalesHistoryAsync()`, `GetExpiredPendingTransactionsAsync()`

**Status**: `GetTransactionStatusByCodeAsync()`

**Validation**: `ValidateTransactionOwnership()`

**Background**: `TransactionAutoConfirmService` (every 10min), `TransactionReservationCleanupService` (every 5min)
