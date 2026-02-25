# EF Core Include/ThenInclude Patterns - TicketHub

## Summary
Found **10 files** with 10+ distinct Include/ThenInclude patterns. Key patterns involve loading transaction items, tickets, events, seats, and related entities.

---

## 1. **TransactionRepository.cs**
**File Path**: `TicketPlatFormServer/Repository/Transaction/TransactionRepository.cs`

### Pattern A: Single Level Include (Status)
```csharp
public async Task<DBModel.Transaction?> GetTransactionById(long transactionId)
{
    return await context.Transactions
        .Where(t => t.Id == transactionId && t.DeletedAt == null)
        .Include(t => t.Status)
        .FirstOrDefaultAsync();
}
```
**Method**: `GetTransactionById(long)`
**Use Case**: Load transaction with its status

---

### Pattern B: Multiple Includes with AsSplitQuery
```csharp
public async Task<DBModel.Transaction?> GetTransactionWithDetailsAsync(long transactionId)
{
    return await context.Transactions
        .Where(t => t.Id == transactionId && t.DeletedAt == null)
        .Include(t => t.Status)
        .Include(t => t.TransactionItems)
        .AsSplitQuery()
        .FirstOrDefaultAsync();
}
```
**Method**: `GetTransactionWithDetailsAsync(long)`
**Includes**: `Status`, `TransactionItems`
**Use Case**: Load transaction with all its items and status (split query prevents Cartesian explosion)

---

### Pattern C: Expired Pending Transactions (Multiple Includes)
```csharp
public async Task<List<DBModel.Transaction>> GetExpiredPendingTransactionsAsync(DateTime utcNow)
{
    return await context.Transactions
        .Where(t => t.DeletedAt == null
                    && t.CancelledAt == null
                    && t.ReservationExpiresAt != null
                    && t.ReservationExpiresAt < utcNow
                    && t.Status.Code == "pending_payment")
        .Include(t => t.Status)
        .Include(t => t.TransactionItems)
        .AsSplitQuery()
        .ToListAsync();
}
```
**Method**: `GetExpiredPendingTransactionsAsync(DateTime)`
**Includes**: `Status`, `TransactionItems`
**Pattern**: Filter by status code in WHERE clause (requires Include for navigation)

---

### Pattern D: Auto-Confirm Transactions
```csharp
public async Task<List<DBModel.Transaction>> GetAutoConfirmDueTransactionsAsync(DateTime utcNow)
{
    return await context.Transactions
        .Where(t => t.DeletedAt == null
                    && t.CancelledAt == null
                    && t.ConfirmedAt == null
                    && t.AutoConfirmAt != null
                    && t.AutoConfirmAt <= utcNow
                    && t.Status.Code == "paid")
        .Include(t => t.Status)
        .AsSplitQuery()
        .ToListAsync();
}
```
**Method**: `GetAutoConfirmDueTransactionsAsync(DateTime)`
**Includes**: `Status`
**Pattern**: Single Include with AsSplitQuery for status filtering

---

## 2. **SellRepository.cs**
**File Path**: `TicketPlatFormServer/Repository/Sell/SellRepository.cs`

### Pattern: Single Include (Ticket Status)
```csharp
public async Task<DBModel.Ticket?> GetTicketByIdAsync(int ticketId)
{
    return await context.Tickets
        .Include(t => t.Status)
        .FirstOrDefaultAsync(t => t.Id == ticketId && t.DeletedAt == null);
}
```
**Method**: `GetTicketByIdAsync(int)`
**Includes**: `Status`
**Use Case**: Load ticket with its status

---

## 3. **PaymentRepository.cs**
**File Path**: `TicketPlatFormServer/Repository/Payment/PaymentRepository.cs`

### Pattern A: Payment with Status, Method, and Transaction
```csharp
public async Task<DBModel.Payment?> GetPaymentByOrderIdAsync(string orderId)
{
    return await context.Payments
        .Where(p => p.OrderId == orderId)
        .Include(p => p.Status)
        .Include(p => p.Method)
        .Include(p => p.Transaction)
        .FirstOrDefaultAsync();
}
```
**Method**: `GetPaymentByOrderIdAsync(string)`
**Includes**: `Status`, `Method`, `Transaction`

---

### Pattern B: Payment by PaymentKey
```csharp
public async Task<DBModel.Payment?> GetPaymentByPaymentKeyAsync(string paymentKey)
{
    return await context.Payments
        .Where(p => p.PaymentKey == paymentKey)
        .Include(p => p.Status)
        .Include(p => p.Method)
        .Include(p => p.Transaction)
        .FirstOrDefaultAsync();
}
```
**Method**: `GetPaymentByPaymentKeyAsync(string)`
**Includes**: `Status`, `Method`, `Transaction`

---

### Pattern C: Payment by TransactionId
```csharp
public async Task<DBModel.Payment?> GetPaymentByTransactionIdAsync(long transactionId)
{
    return await context.Payments
        .Where(p => p.TransactionId == transactionId)
        .Include(p => p.Status)
        .Include(p => p.Method)
        .Include(p => p.Transaction)
        .FirstOrDefaultAsync();
}
```
**Method**: `GetPaymentByTransactionIdAsync(long)`
**Includes**: `Status`, `Method`, `Transaction`

---

### Pattern D: Escrow with Status and Transaction
```csharp
public async Task<Escrow?> GetEscrowByTransactionIdAsync(long transactionId)
{
    return await context.Escrows
        .Where(e => e.TransactionId == transactionId)
        .Include(e => e.Status)
        .Include(e => e.Transaction)
        .FirstOrDefaultAsync();
}
```
**Method**: `GetEscrowByTransactionIdAsync(long)`
**Includes**: `Status`, `Transaction`

---

## 4. **ChatRepository.cs**
**File Path**: `TicketPlatFormServer/Repository/Chat/ChatRepository.cs`

### Pattern A: ChatRoom by Id (Deep Chain)
```csharp
public async Task<ChatRoom?> GetChatRoomById(long roomId)
{
    return await db.ChatRooms
        .AsNoTracking()
        .Include(cr => cr.Ticket).ThenInclude(t => t!.Event)
        .Include(cr => cr.Ticket).ThenInclude(t => t!.SeatGrade)
        .Include(cr => cr.Ticket).ThenInclude(t => t!.SeatLocation)
        .Include(cr => cr.Ticket).ThenInclude(t => t!.Area)
        .Include(cr => cr.Buyer).ThenInclude(u => u.UserProfile)
        .Include(cr => cr.Seller).ThenInclude(u => u.UserProfile)
        .Include(cr => cr.Status)
        .Include(cr => cr.Transaction).ThenInclude(t => t!.Status)
        .FirstOrDefaultAsync(cr => cr.Id == roomId && cr.DeletedAt == null);
}
```
**Method**: `GetChatRoomById(long)`
**Include Chain**:
- `Ticket` → `Event`
- `Ticket` → `SeatGrade`
- `Ticket` → `SeatLocation`
- `Ticket` → `Area`
- `Buyer` → `UserProfile`
- `Seller` → `UserProfile`
- `Status`
- `Transaction` → `Status`

**Pattern Notes**:
- Multiple `ThenInclude()` on same parent using repeated `.Include(cr => cr.Ticket)`
- Uses `AsNoTracking()` for read-only optimization
- Nullable forgiving operator `!` in lambda
- Deep nesting (2 levels)

---

### Pattern B: ChatRoom by TransactionId (Same as Pattern A)
```csharp
public async Task<ChatRoom?> GetChatRoomByTransactionId(long transactionId)
{
    return await db.ChatRooms
        .AsNoTracking()
        .Include(cr => cr.Ticket).ThenInclude(t => t!.Event)
        .Include(cr => cr.Ticket).ThenInclude(t => t!.SeatGrade)
        .Include(cr => cr.Ticket).ThenInclude(t => t!.SeatLocation)
        .Include(cr => cr.Ticket).ThenInclude(t => t!.Area)
        .Include(cr => cr.Buyer).ThenInclude(u => u.UserProfile)
        .Include(cr => cr.Seller).ThenInclude(u => u.UserProfile)
        .Include(cr => cr.Status)
        .Include(cr => cr.Transaction).ThenInclude(t => t!.Status)
        .FirstOrDefaultAsync(cr => cr.TransactionId == transactionId && cr.DeletedAt == null);
}
```
**Method**: `GetChatRoomByTransactionId(long)`
**Identical includes** to Pattern A

---

### Pattern C: ChatRoom by TicketId and BuyerId (Same Structure)
```csharp
public async Task<ChatRoom?> GetChatRoomByTicketAndBuyer(int ticketId, int buyerId)
{
    return await db.ChatRooms
        .AsNoTracking()
        .Include(cr => cr.Ticket).ThenInclude(t => t!.Event)
        .Include(cr => cr.Ticket).ThenInclude(t => t!.SeatGrade)
        .Include(cr => cr.Ticket).ThenInclude(t => t!.SeatLocation)
        .Include(cr => cr.Ticket).ThenInclude(t => t!.Area)
        .Include(cr => cr.Buyer).ThenInclude(u => u.UserProfile)
        .Include(cr => cr.Seller).ThenInclude(u => u.UserProfile)
        .Include(cr => cr.Status)
        .Include(cr => cr.Transaction).ThenInclude(t => t!.Status)
        .FirstOrDefaultAsync(cr => cr.TicketId == ticketId && cr.BuyerId == buyerId && cr.DeletedAt == null);
}
```
**Method**: `GetChatRoomByTicketAndBuyer(int, int)`
**Same Include chain** as Pattern A/B

---

### Pattern D: ChatRoom by TicketId and UserId (Same Structure)
```csharp
public async Task<ChatRoom?> GetChatRoomByTicketAndUser(int ticketId, int userId)
{
    return await db.ChatRooms
        .AsNoTracking()
        .Include(cr => cr.Ticket).ThenInclude(t => t!.Event)
        .Include(cr => cr.Ticket).ThenInclude(t => t!.SeatGrade)
        .Include(cr => cr.Ticket).ThenInclude(t => t!.SeatLocation)
        .Include(cr => cr.Ticket).ThenInclude(t => t!.Area)
        .Include(cr => cr.Buyer).ThenInclude(u => u.UserProfile)
        .Include(cr => cr.Seller).ThenInclude(u => u.UserProfile)
        .Include(cr => cr.Status)
        .Include(cr => cr.Transaction).ThenInclude(t => t!.Status)
        .FirstOrDefaultAsync(cr =>
            cr.TicketId == ticketId &&
            (cr.BuyerId == userId || cr.SellerId == userId) &&
            cr.DeletedAt == null);
}
```
**Method**: `GetChatRoomByTicketAndUser(int, int)`
**Same Include chain** as Pattern A/B/C

---

### Pattern E: ChatRooms by UserId (Paginated, Partial Includes)
```csharp
public async Task<List<ChatRoom>> GetChatRoomsByUserId(int userId, int page, int pageSize)
{
    var offset = (page - 1) * pageSize;

    var rooms = await db.ChatRooms
        .AsNoTracking()
        .Include(cr => cr.Ticket).ThenInclude(t => t!.Event)
        .Include(cr => cr.Buyer).ThenInclude(u => u.UserProfile)
        .Include(cr => cr.Seller).ThenInclude(u => u.UserProfile)
        .Include(cr => cr.Status)
        .Include(cr => cr.Transaction).ThenInclude(t => t!.Status)
        .Where(cr => (cr.BuyerId == userId || cr.SellerId == userId)
                  && cr.DeletedAt == null
                  && cr.ClosedAt == null
                  && cr.LastMessageAt != null)
        .OrderByDescending(cr => cr.LastMessageAt)
        .Skip(offset)
        .Take(pageSize)
        .ToListAsync();

    return rooms;
}
```
**Method**: `GetChatRoomsByUserId(int, int, int)`
**Includes** (subset of full pattern):
- `Ticket` → `Event` (only Event, not SeatGrade/SeatLocation/Area)
- `Buyer` → `UserProfile`
- `Seller` → `UserProfile`
- `Status`
- `Transaction` → `Status`

**Pattern Notes**:
- Pagination with Skip/Take
- OrderByDescending added
- Missing seat-related includes (optimization for list view)

---

## 5. **UserRepository.cs**
**File Path**: `TicketPlatFormServer/Repository/User/UserRepository.cs`

### Pattern A: Get User by Email
```csharp
public async Task<User?> GetByEmail(string email)
{
    var user = await db.Users
        .Include(u => u.Provider)
        .Include(u => u.Role)
        .FirstOrDefaultAsync(x => x.Email == email && x.IsDeleted == false);
    return user;
}
```
**Method**: `GetByEmail(string)`
**Includes**: `Provider`, `Role`

---

### Pattern B: Get User by Id
```csharp
public async Task<User?> GetByIdAsync(int userId)
{
    var user = await db.Users
        .Include(u => u.Provider)
        .Include(u => u.Role)
        .FirstOrDefaultAsync(x => x.Id == userId && x.IsDeleted == false);
    return user;
}
```
**Method**: `GetByIdAsync(int)`
**Includes**: `Provider`, `Role`

---

## 6. **DisputeRepository.cs**
**File Path**: `TicketPlatFormServer/Repository/Dispute/DisputeRepository.cs`

### Pattern: Dispute with Details (Deep Chain)
```csharp
public async Task<Dispute?> GetDisputeByIdWithDetailsAsync(long disputeId)
{
    return await context.Disputes
        .Include(x => x.Type)
        .Include(x => x.Status)
        .Include(x => x.DisputeEvidences)
        .Include(x => x.Transaction)
            .ThenInclude(t => t.TransactionItems)
        .AsSplitQuery()
        .FirstOrDefaultAsync(x => x.Id == disputeId);
}
```
**Method**: `GetDisputeByIdWithDetailsAsync(long)`
**Include Chain**:
- `Type`
- `Status`
- `DisputeEvidences`
- `Transaction` → `TransactionItems`

**Pattern Notes**:
- Chained `.ThenInclude()` after `.Include(x => x.Transaction)`
- Uses `AsSplitQuery()` to prevent Cartesian explosion
- Loads full transaction details including items

---

### Pattern B: Disputes by Claimant (Cursor Pagination)
```csharp
public async Task<List<Dispute>> GetDisputesByClaimantCursorAsync(long claimantId, long? cursorId, int limitPlusOne)
{
    var query = context.Disputes
        .AsNoTracking()
        .Include(x => x.Type)
        .Include(x => x.Status)
        .Where(x => x.ClaimantId == claimantId);

    if (cursorId.HasValue)
    {
        query = query.Where(x => x.Id < cursorId.Value);
    }

    return await query
        .OrderByDescending(x => x.Id)
        .Take(limitPlusOne)
        .ToListAsync();
}
```
**Method**: `GetDisputesByClaimantCursorAsync(long, long?, int)`
**Includes**: `Type`, `Status`
**Pattern Notes**:
- Uses `AsNoTracking()` for read-only list
- Cursor-based pagination (no TransactionItems loaded)
- Conditional Where clause for cursor

---

## Key Patterns Summary

| Pattern | Usage | EF Features |
|---------|-------|----------|
| **Single Include** | Load one navigation property | `.Include(t => t.Status)` |
| **Multiple Includes** | Load several independent navigation properties | Multiple `.Include()` calls |
| **ThenInclude Chain** | Load nested relationships (2+ levels) | `.Include().ThenInclude()` |
| **AsSplitQuery** | Prevent Cartesian explosion with multiple collections | `.AsSplitQuery()` |
| **AsNoTracking** | Optimize read-only queries | `.AsNoTracking()` |
| **Conditional Includes** | Include based on logic | Query variables with conditional `.Include()` |

---

## Query Optimization Techniques Used

1. **AsSplitQuery()** - Used in TransactionRepository and DisputeRepository when loading multiple collections
2. **AsNoTracking()** - Used in ChatRepository and DisputeRepository for read-only queries
3. **Repeated Include Syntax** - ChatRepository uses `.Include(cr => cr.Ticket).ThenInclude(...)` multiple times for same parent
4. **Selective Loading** - ChatRepository's list method loads fewer properties than detail methods
5. **Pagination with Skip/Take** - ChatRepository and SellRepository apply pagination after includes

---

## Recommendations for New Include Patterns

1. **For Transaction-heavy queries**: Follow TransactionRepository pattern with `.AsSplitQuery()` when loading multiple collections
2. **For Chat/Room queries**: Use ChatRepository as template with deep ThenInclude chains
3. **For Status-dependent filtering**: Always include status navigation before filtering by `.Status.Code`
4. **For List endpoints**: Use `AsNoTracking()` + selective includes (only what's needed for list view)
5. **For Detail endpoints**: Use full include chains with `AsSplitQuery()` to load all related data
