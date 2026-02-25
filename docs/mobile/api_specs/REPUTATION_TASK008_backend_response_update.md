# Reputation API and DTO Changes (TASK-008)

## New APIs

- `POST /api/reputations`
  - Auth required
  - Request: `transactionId`, `score(1..5)`
  - Response: `ApiResponse<long>` (created reputation id)
- `GET /api/reputations/check/{transactionId}`
  - Auth required
  - Response: `ApiResponse<ReputationCheckRespDto>`
- `GET /api/users/{userId}/reputations?page=1&size=20`
  - Public
  - Response: `ApiResponse<ReputationListRespDto>`

## Added DTOs

- `CreateReputationReqDto`
  - `transactionId: long`
  - `score: int` (1~5)
- `ReputationCheckRespDto`
  - `canReview: bool`
  - `hasReviewed: bool`
  - `reviewDeadline?: string` (ISO8601, `null` when expired)
- `ReputationRespDto`
  - `id: long`
  - `reviewerNickname: string`
  - `reviewerProfileImageUrl?: string`
  - `score: int`
  - `createdAt: string` (ISO8601)
- `ReputationListRespDto`
  - `items: ReputationRespDto[]`
  - `totalCount: int`
  - `averageRating?: float`

## Existing DTO Field Additions

- `UserProfileDto`
  - `averageRating?: float` added
  - `reviewCount: int` added
- `ChatRoomDetailRespDto`
  - `canWriteReview: bool` added
  - `hasReviewedSeller: bool` added

## Notes for Mobile

- `canWriteReview` becomes `true` only if all conditions pass:
  - current user is buyer
  - transaction is confirmed and not cancelled
  - within 7 days from `confirmedAt`
  - buyer has not already reviewed
  - no open dispute (`PENDING` or `IN_REVIEW`)
- `REVIEW_REQUEST` notification type is now produced on manual and automatic purchase confirmation flows.
