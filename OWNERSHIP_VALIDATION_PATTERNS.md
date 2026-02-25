# Ownership Validation Patterns in TicketHub

## Overview
This document captures the authorization/ownership validation patterns used throughout the TicketHub codebase. All validation follows a consistent pattern: **Check for existence first (404), then check ownership (403)**.

---

## Core Pattern

### HTTP Status Code Convention
- **404 NotFound** - Resource doesn't exist (check first)
- **403 Forbidden** - Resource exists but user lacks access (check second)

---

## Implementation Patterns

### Pattern 1: Direct Ownership Check (Most Common)

**Location**: Services that need to verify a user owns a specific entity

**Structure**:
```csharp
// 1. Get the resource (Entity must include user ID reference)
var resource = await repository.GetResourceByIdAsync(resourceId);
if (resource == null)
{
    throw new AppException("리소스를 찾을 수 없습니다.", HttpStatusCode.NotFound);
}

// 2. Check ownership
if (resource.UserId != userId)  // Or SellerId, BuyerId, ClaimantId
{
    throw new AppException("접근할 권한이 없습니다.", HttpStatusCode.Forbidden);
}
```

---

## Real-World Examples

### 1. **Ticket Ownership** (SellService.cs)

**File**: `Services/Sell/SellService.cs`

```csharp
/// Refresh ticket images - seller only
public async Task<RefreshTicketImageUrlRespDto> RefreshTicketImageUrlsAsync(int ticketId, int userId)
{
    // 1. Ticket 조회 및 소유권 확인
    var ticket = await _sellRepository.GetTicketByIdAsync(ticketId);
    if (ticket == null)
    {
        throw new AppException("티켓을 찾을 수 없습니다.", HttpStatusCode.NotFound);
    }

    // 2. Ownership check
    if (ticket.SellerId != userId)
    {
        throw new AppException("티켓에 접근할 권한이 없습니다.", HttpStatusCode.Forbidden);
    }

    // ... rest of logic
}
```

**Pattern Notes**:
- Checks `SellerId` field (entity has this reference)
- 404 first, then 403
- Used for: Cancel ticket, refresh images, etc.

---

### 2. **Transaction Ownership - Buyer Perspective** (PaymentService.cs)

**File**: `Services/Payment/PaymentService.cs`

```csharp
/// Payment initiation - buyer only
public async Task<PaymentRequestResponseDto> InitiatePaymentAsync(PaymentRequestDto request, int userId)
{
    logger.LogInformation("[PaymentService.InitiatePaymentAsync] TransactionId: {TransactionId}, UserId: {UserId}",
        request.TransactionId, userId);

    // 1. Transaction 존재 및 소유권 검증
    var transaction = await transactionRepository.GetTransactionById(request.TransactionId);
    if (transaction == null)
    {
        throw new AppException("거래를 찾을 수 없습니다.", HttpStatusCode.NotFound);
    }

    // ... status validation ...

    // 2. Buyer ownership check
    if (transaction.BuyerId != userId)
    {
        throw new AppException("해당 거래의 구매자만 결제를 요청할 수 있습니다.", HttpStatusCode.Forbidden);
    }

    // ... rest of logic
}
```

**Pattern Notes**:
- Checks `BuyerId` field for buyer-specific actions
- Includes business logic validation between existence and ownership checks
- Used for: Payment initiation, purchase confirmation, etc.

---

### 3. **Role-Based Ownership Check** (DisputeService.cs)

**File**: `Services/Dispute/DisputeService.cs`

```csharp
/// Dispute creation - buyer only
public async Task<DisputeSummaryRespDto> CreateDisputeAsync(long userId, CreateDisputeReqDto req)
{
    var transaction = await transactionRepository.GetTransactionById(req.TransactionId);
    if (transaction == null)
    {
        throw new AppException("거래가 존재하지 않습니다.", HttpStatusCode.NotFound);
    }

    // Buyer ownership check
    if (transaction.BuyerId != userId)
    {
        throw new AppException("해당 거래의 구매자만 신고할 수 있습니다.", HttpStatusCode.Forbidden);
    }

    // ... more validation ...
}
```

**Get disputes with role differentiation**:
```csharp
public async Task<DisputeListRespDto> GetMyDisputesAsync(long userId, string? cursor, int? limit)
{
    var disputes = await disputeRepository.GetDisputesByUserIdAsync(userId, cursorId, actualLimit + 1);
    
    var items = disputes.Select(x => new DisputeItemRespDto
    {
        Id = x.Id,
        TransactionId = x.TransactionId,
        // ... other fields
        
        // Determine role within dispute
        IsBuyer = x.Transaction.BuyerId == userId,
        IsSeller = x.Transaction.SellerId == userId,
    }).ToList();
}
```

**Pattern Notes**:
- Single role check: `if (transaction.BuyerId != userId)`
- Multiple role check: `IsBuyer = x.BuyerId == userId; IsSeller = x.SellerId == userId;`
- Used for: Disputes, transaction history, role-based actions

---

### 4. **Dual Party Authorization** (ChatService.cs - Multiple Examples)

**File**: `Services/Chat/ChatService.cs`

#### Example 4A: Buyer-only action
```csharp
/// Confirm purchase - buyer only
public async Task<PurchaseConfirmRespDto> ConfirmPurchase(ConfirmPurchaseReqDto req)
{
    var room = await chatRepo.GetChatRoomById(req.RoomId);
    if (room == null)
    {
        throw new AppException("채팅방을 찾을 수 없습니다.", HttpStatusCode.NotFound);
    }

    // 구매자인지 확인 (role-specific)
    if (room.BuyerId != req.UserId)
    {
        throw new AppException("구매자만 구매를 확정할 수 있습니다.", HttpStatusCode.Forbidden);
    }

    // ... rest of logic
}
```

#### Example 4B: Seller-only action
```csharp
/// Request payment - seller only
public async Task<PaymentUrlRespDto> RequestPayment(long roomId, int userId, int quantity)
{
    // First check if user is in room (general access)
    await ValidateUserInRoom(roomId, userId);

    var room = await chatRepo.GetChatRoomById(roomId);
    if (room == null)
    {
        throw new AppException("채팅방을 찾을 수 없습니다.", HttpStatusCode.NotFound);
    }

    // 판매자인지 확인 (role-specific)
    if (room.SellerId != userId)
    {
        throw new AppException("판매자만 결제를 요청할 수 있습니다.", HttpStatusCode.Forbidden);
    }

    // ... rest of logic
}
```

#### Example 4C: Either party can cancel
```csharp
/// Cancel transaction - either party
public async Task CancelTransaction(CancelTransactionReqDto req)
{
    // First check if user is in room
    await ValidateUserInRoom(req.RoomId, req.UserId);

    var room = await chatRepo.GetChatRoomById(req.RoomId);
    if (room == null)
    {
        throw new AppException("채팅방을 찾을 수 없습니다.", HttpStatusCode.NotFound);
    }

    // 구매자 또는 판매자인지 확인
    if (room.BuyerId != req.UserId && room.SellerId != req.UserId)
    {
        throw new AppException("거래 당사자만 취소할 수 있습니다.", HttpStatusCode.Forbidden);
    }

    // ... rest of logic
}
```

**Pattern Notes**:
- Buyer/Seller determined by `BuyerId`/`SellerId` fields
- Dual-party actions use OR logic: `(room.BuyerId != userId && room.SellerId != userId)`
- Used for: Payment requests, confirmations, cancellations

---

### 5. **Helper Validation Method** (ChatService.cs)

**File**: `Services/Chat/ChatService.cs`

```csharp
/// Reusable validation helper
private async Task ValidateUserInRoom(long roomId, int userId)
{
    var isInRoom = await chatRepo.IsUserInChatRoom(roomId, userId);
    if (!isInRoom)
    {
        throw new AppException("이 채팅방에 접근할 권한이 없습니다.", HttpStatusCode.Forbidden);
    }
}

// Usage - called before role-specific checks
public async Task MarkMessagesAsRead(long roomId, int userId)
{
    // Check existence first
    var room = await chatRepo.GetChatRoomById(roomId);
    if (room == null)
    {
        throw new AppException("채팅방을 찾을 수 없습니다.", HttpStatusCode.NotFound);
    }

    // Then check general access
    await ValidateUserInRoom(roomId, userId);

    // Then do role-specific logic
    var isBuyer = room.BuyerId == userId;
    await chatRepo.ResetUnreadCount(roomId, isBuyer);
}
```

**Pattern Notes**:
- Extracted to private helper method
- Always called **after** existence check, **before** role-specific logic
- Prevents code duplication across 10+ methods
- Uses: `IsUserInChatRoom()` repository query

---

### 6. **Notification Ownership** (NotificationService.cs)

**File**: `Services/Notification/NotificationService.cs`

```csharp
/// Mark notification as read - user owns it
public async Task MarkAsReadAsync(long userId, long notificationId)
{
    var target = await notificationRepository.GetByIdAsync(notificationId);
    if (target == null)
    {
        throw new AppException("알림이 존재하지 않습니다.", HttpStatusCode.NotFound);
    }

    // User ownership check
    if (target.UserId != userId)
    {
        throw new AppException("본인의 알림이 아닙니다.", HttpStatusCode.Forbidden);
    }

    if (target.ReadFlag == true)
    {
        return;
    }

    await notificationRepository.MarkAsReadAsync(notificationId, DateTime.UtcNow);
}
```

**Pattern Notes**:
- Direct `UserId` check
- Idempotent: returns early if already read
- Used for: Notifications, profile data, personal settings

---

## Summary Table

| Entity | Field | Check | Pattern | File |
|--------|-------|-------|---------|------|
| **Ticket** | `SellerId` | Seller owns ticket | Direct field check | SellService.cs |
| **Transaction** | `BuyerId` | Buyer owns transaction | Direct field check | PaymentService.cs |
| **Transaction** | `BuyerId`/`SellerId` | Either party | OR logic check | DisputeService.cs |
| **ChatRoom** | `BuyerId`/`SellerId` | Party in room | OR logic or helper | ChatService.cs |
| **Notification** | `UserId` | User owns notification | Direct field check | NotificationService.cs |
| **ChatRoom** | General access | User is in room | Helper method | ChatService.cs |

---

## Key Rules to Follow

1. **Always 404 before 403**
   ```csharp
   // ❌ WRONG: 403 before 404
   if (entity.UserId != userId) throw Forbidden();
   if (entity == null) throw NotFound();
   
   // ✅ CORRECT: 404 before 403
   if (entity == null) throw NotFound();
   if (entity.UserId != userId) throw Forbidden();
   ```

2. **Use appropriate field names**
   - `UserId` - when entity directly belongs to a user
   - `SellerId` - when checking seller ownership
   - `BuyerId` - when checking buyer ownership
   - `ClaimantId` - when checking dispute reporter

3. **Extract reusable validation to helpers**
   - Use `private async Task Validate...()` methods
   - Called after existence checks
   - Prevents duplication across 5+ uses

4. **Use OR logic for dual-party actions**
   ```csharp
   if (room.BuyerId != userId && room.SellerId != userId)
   {
       throw new AppException("당사자만...", HttpStatusCode.Forbidden);
   }
   ```

5. **Include context in error messages**
   ```csharp
   // ✅ Clear who can do what
   "해당 거래의 구매자만 결제를 요청할 수 있습니다."
   "판매자만 결제를 요청할 수 있습니다."
   
   // ❌ Vague
   "권한이 없습니다."
   ```

---

## Mapping to DTOs

When returning data to controllers, determine what user can see:

```csharp
var items = disputes.Select(x => new DisputeItemRespDto
{
    Id = x.Id,
    TransactionId = x.TransactionId,
    
    // Determine role (for conditional UI)
    IsBuyer = x.Transaction.BuyerId == userId,
    IsSeller = x.Transaction.SellerId == userId,
    
    // Filter sensitive data based on role
    BuyerName = x.Transaction.BuyerId == userId ? x.Transaction.Buyer.Name : null,
    SellerName = x.Transaction.SellerId == userId ? x.Transaction.Seller.Name : null,
}).ToList();
```

---

## Anti-Patterns to Avoid

❌ **Mixed with business logic**
```csharp
// DON'T do validation and business logic at same time
if (ticket == null || ticket.SellerId != userId) { /* error */ }
```

❌ **Checking by name instead of ID**
```csharp
// DON'T: unreliable, slow
if (ticket.Seller.Name != currentUserName) { /* error */ }
```

❌ **No existence check**
```csharp
// DON'T: will throw different error if null
var ticket = await repo.GetAsync(id);  // Could be null
if (ticket.SellerId != userId) { /* error */ }  // NullRef
```

---

## Real Implementation Checklist

When adding a new authorization check:

- [ ] Fetch entity from repository
- [ ] Check `== null` first (404 NotFound)
- [ ] Check ownership field (403 Forbidden)
- [ ] Use role-appropriate field names (SellerId, BuyerId, UserId)
- [ ] Include context in error message ("해당 거래의 구매자만...")
- [ ] Use appropriate HTTP status codes (404, 403)
- [ ] Extract to helper if reused 2+ times
- [ ] Add logging (include UserId and EntityId)