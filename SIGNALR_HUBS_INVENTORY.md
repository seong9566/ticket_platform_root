# SignalR Hub Implementation Inventory

## Overview
The TicketHub platform implements real-time communication using SignalR, primarily focused on chat notifications and transaction updates. All hubs inherit from `Hub` base class with JWT authentication enforced.

---

## Hub Implementation

### 1. **ChatHub** 
**Location**: `Hubs/ChatHub.cs`  
**Authorization**: `[Authorize]` (JWT required)  
**Dependencies**: 
- `IChatRepository` - for permission validation
- `ILogger<ChatHub>` - structured logging

#### Methods:

| Method | Type | Notification Type | Target Group | Purpose |
|--------|------|-------------------|---------------|---------|
| `OnConnectedAsync()` | Lifecycle | Connection | N/A | Adds user to personal group `user_{userId}` on connection |
| `OnDisconnectedAsync()` | Lifecycle | Disconnection | N/A | Logs disconnection events (user group removed automatically) |
| `JoinRoom(long roomId)` | Hub method | UserJoined | `room_{roomId}` (others only) | User enters chat room, broadcasts to other room participants |
| `LeaveRoom(long roomId)` | Hub method | UserLeft | `room_{roomId}` (others only) | User exits chat room, broadcasts to other room participants |
| `UserTyping(long roomId)` | Hub method | UserTyping | `room_{roomId}` (others only) | Real-time typing indicator, broadcasts to same room only |
| `UserStoppedTyping(long roomId)` | Hub method | UserStoppedTyping | `room_{roomId}` (others only) | Typing stopped, broadcasts to same room only |

#### Client Invocation Examples:
```csharp
// Room-based broadcast (others only)
await Clients.OthersInGroup($"room_{roomId}").SendAsync("UserJoined", new {
    UserId = userId.Value,
    RoomId = roomId,
    Timestamp = DateTime.UtcNow
});

// User-specific broadcast (all user's connections)
await Clients.Group($"user_{userId}").SendAsync("NewMessage", signalDto);
```

---

## Service-Level SignalR Integration

Services inject `IHubContext<ChatHub>` to broadcast notifications outside the hub context.

### 2. **ChatService**
**Location**: `Services/Chat/ChatService.cs`  
**IHubContext**: `IHubContext<ChatHub> hubContext`

#### Broadcasting Methods:

| Service Method | Client Event | Target Group | Notification |
|---|---|---|---|
| `SendMessageAsync()` | `ReceiveMessage` | `room_{roomId}`, `user_{buyerId}`, `user_{sellerId}` | New chat message |
| `RequestPayment()` | `RoomUpdated` | `room_{roomId}` | Payment requested |
| `ConfirmPurchase()` | `ReceiveMessage`, `NewMessage` | `room_{roomId}`, `user_{buyerId}`, `user_{sellerId}` | Purchase confirmed (system message) |
| `CancelTransaction()` | `ReceiveMessage` | `room_{roomId}`, `user_{receiverId}` | Transaction cancelled |
| `LeaveChatRoom()` | `ReceiveMessage` | `room_{roomId}`, `user_{receiverId}` | User left chat room |

**Notification DTO**: `NewMessageSignalDto`
- MessageId, RoomId, SenderId, SenderNickname, SenderProfileImage
- Message, Type (enum: TEXT, SYSTEM, PURCHASE_CONFIRMED, etc.)
- CreatedAt

### 3. **PaymentService**
**Location**: `Services/Payment/PaymentService.cs`  
**IHubContext**: `IHubContext<ChatHub> hubContext`

#### Broadcasting Methods:

| Service Method | Client Event | Target Group | Notification |
|---|---|---|---|
| Payment Success Handler | `ReceiveMessage` | `room_{room.Id}`, `user_{room.BuyerId}`, `user_{room.SellerId}` | PAYMENT_SUCCESS (system message) |

---

## Controller-Level SignalR Integration

### 4. **ChatController**
**Location**: `Controllers/ChatController.cs`  
**IHubContext**: `IHubContext<ChatHub> hubContext` (injected in constructor)

#### Broadcasting Methods:

| Endpoint | Method | Client Event | Target Group | Purpose |
|---|---|---|---|---|
| `POST /messages` | SendMessage | `ReceiveMessage` | `user_{senderId}`, `user_{receiverId}` | Message delivery confirmation to both parties |
| `POST /rooms/request-payment` | RequestPayment | `RoomUpdated` | `room_{roomId}` | Payment request notification to room |
| `POST /rooms/confirm-purchase` | ConfirmPurchase | `RoomUpdated` | `room_{roomId}` | Purchase confirmation to room |
| `POST /rooms/cancel-transaction` | CancelTransaction | `RoomUpdated` | `room_{roomId}` | Transaction cancellation to room |

**Notification DTO**: `RoomUpdatedSignalDto`
- RoomId, Event (string: "PaymentRequested", "TransactionCancelled", etc.)
- TransactionId, StatusCode, Message

---

## Group Architecture

### User-Level Groups
- **Format**: `user_{userId}`
- **Members**: All connections belonging to the user
- **Use Case**: User-specific notifications (messages, alerts)
- **Scope**: User-wide, persists across room joins

### Room-Level Groups
- **Format**: `room_{roomId}`
- **Members**: All connections in a specific chat room
- **Use Case**: Room-specific broadcasts (typing indicators, messages)
- **Scope**: Room-specific, cleaned up on `LeaveRoom()`

### Broadcast Patterns
| Pattern | Use | Example |
|---------|-----|---------|
| `Clients.All` | ❌ Not used | (broadcast to all users) |
| `Clients.Group($"room_{roomId}")` | Room-wide updates | Payment request, transaction cancellation |
| `Clients.Group($"user_{userId}")` | User-specific notifications | New message for user |
| `Clients.OthersInGroup($"room_{roomId}")` | Exclude self | Typing indicators, join/leave events |
| `Clients.Caller` | ❌ Not used | (not needed in this design) |

---

## Message Types (Enums)

**MessageType** enum used in `NewMessageSignalDto`:
- `TEXT` - Regular chat message
- `SYSTEM` - Internal system message
- `PURCHASE_CONFIRMED` - Purchase confirmation system message
- `PAYMENT_SUCCESS` - Payment successful system message

---

## Security Patterns

### Authentication
- All hubs require `[Authorize]` attribute
- JWT token validated on connection
- UserId extracted from JWT claims: `Context.User?.GetUserId()`

### Authorization
- Room access validated before joining/leaving
  ```csharp
  var isInRoom = await chatRepo.IsUserInChatRoom(roomId, userId.Value);
  if (!isInRoom) throw new HubException("접근 권한이 없습니다.");
  ```
- Only room participants can receive room-level notifications

### Error Handling
- **Hub errors**: Throw `HubException` (client receives `InvocationError`)
- **Service errors**: Throw `AppException` (handled by controller/middleware)
- Detailed logging with context (userId, roomId, connectionId, timestamp)

---

## Real-Time Notification Flow

### Chat Message Flow
```
Client -> ChatController.SendMessage() 
  -> ChatService.SendMessageAsync() 
  -> DB save 
  -> IHubContext.Clients.Group().SendAsync("ReceiveMessage")
  -> Both users receive update
```

### Payment Transaction Flow
```
Client -> ChatController.RequestPayment() 
  -> ChatService.RequestPayment() 
  -> Toss payment gateway 
  -> IHubContext.Clients.Group().SendAsync("RoomUpdated")
  -> Room participants notified
```

### Typing Indicator Flow
```
Client -> ChatHub.UserTyping(roomId) 
  -> Validation + permission check 
  -> IHubContext.Clients.OthersInGroup().SendAsync("UserTyping")
  -> Other room members see typing indicator
```

---

## Connected Services

### Notification Service Integration
- `NotificationService` creates and sends FCM push notifications
- Called alongside SignalR for critical events (payment, transaction confirmation)
- Ensures users receive notifications even when not connected to SignalR

**Example** (PaymentService):
```csharp
// SignalR real-time notification
await hubContext.Clients.Group($"user_{room.BuyerId}")
    .SendAsync("ReceiveMessage", messageDto);

// FCM push notification (backup)
await notificationService.CreateAndSendAsync(
    room.BuyerId,
    "PAYMENT_SUCCESS",
    "결제가 완료되었습니다",
    "결제가 정상적으로 완료되었습니다.",
    new Dictionary<string, string> {
        ["type"] = "PAYMENT_SUCCESS",
        ["transactionId"] = transactionId.ToString()
    });
```

---

## Implementation Statistics

| Metric | Count |
|--------|-------|
| **Hub Classes** | 1 (ChatHub) |
| **Hub Methods** | 6 (including lifecycle) |
| **Client Events** | 6 unique |
| **Service Files Using IHubContext** | 3 (ChatService, PaymentService, ChatController) |
| **Total SendAsync Invocations** | 17 across codebase |
| **Group-based Broadcasts** | Room + User groups |
| **Authentication** | JWT via [Authorize] |

---

## Future Extension Points

1. **Notification Hub** - Could add separate `NotificationHub` for general notifications
2. **Dispute Updates** - `DisputeService` calls `notificationService` but could integrate SignalR for real-time updates
3. **Admin Broadcasts** - Could add admin-only hub methods for server announcements
4. **Transaction Status Hub** - Real-time payment processing updates

---

## Configuration Requirements

SignalR is configured in `Program.cs`:
- Endpoint: `/hub/chat` (inferred from ChatHub routing)
- Authentication: JWT Bearer token required
- CORS: Configured for allowed origins
- Connection timeout: Standard ASP.NET Core defaults
