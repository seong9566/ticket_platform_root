# Flutter Handoff: Home `deadlineDeals` API Contract

## Change Summary

- Endpoint: `GET /api/home`
- Added response field: `data.deadlineDeals`
- Existing fields remain: `data.popularEvents`, `data.recommendedEvents`

## Backend DTO Changes

- Added `TicketPlatFormServer/TicketPlatFormServer/DTO/Home/DeadlineDealDto.cs`
- Updated `TicketPlatFormServer/TicketPlatFormServer/DTO/Home/HomeRespDto.cs`
  - Added `DeadlineDeals: List<DeadlineDealDto> = new()`

## Required JSON Shape (Mobile Contract)

```json
{
  "statusCode": 200,
  "message": "홈 화면 데이터 조회 성공",
  "success": true,
  "data": {
    "deadlineDeals": [
      {
        "eventId": 15,
        "eventTitle": "BTS Yet To Come 부산 콘서트",
        "eventDate": "2026.02.25",
        "venue": "부산 아시아드 주경기장",
        "daysLeft": 2,
        "minTicketPrice": 85000,
        "originalMinTicketPrice": 132000,
        "ticketDiscountRate": 35,
        "posterImageUrl": "https://picsum.photos/400/600?random=15",
        "availableTicketCount": 12,
        "categoryId": 1
      }
    ],
    "popularEvents": [],
    "recommendedEvents": []
  }
}
```

## Field Rules

- Key name must be camelCase: `deadlineDeals`
- Numeric fields must be JSON numbers (not strings):
  - `eventId`, `daysLeft`, `minTicketPrice`, `originalMinTicketPrice`, `ticketDiscountRate`, `availableTicketCount`, `categoryId`
- `popularEvents` and `recommendedEvents` must always exist (at least `[]`)

## Backend Query/Service Wiring

- Added repository method: `GetDeadlineDeals(int limit = 10)`
- Added SQL: D-3 filter + selling tickets + sort by discount/day/count
- Home service now includes `deadlineDeals` in `HomeRespDto`

## Quick Verification

```bash
curl -s http://localhost:5224/api/home | jq '.data.deadlineDeals'
curl -s http://localhost:5224/api/home | jq '.data | {deadlineDeals, popularEvents, recommendedEvents}'
```
