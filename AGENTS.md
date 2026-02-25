# PROJECT KNOWLEDGE BASE

**Generated:** 2026-02-25 14:47 KST
**Commit:** 7d1c4b8
**Branch:** feat/notification-fcm

## OVERVIEW
Monorepo with two active products: ASP.NET Core 9 backend (`TicketPlatFormServer`) and Flutter mobile app (`ticket_platform_mobile`).
Root does orchestration and docs; most code lives in project subdirectories.

## STRUCTURE
```text
./
├── TicketPlatFormServer/        # .NET solution root, backend guidance
│   └── TicketPlatFormServer/    # API project (Program.cs, Controllers, Services, Repository)
├── ticket_platform_mobile/      # Flutter app (feature-first Clean MVVM)
├── docs/                        # PM/backend/mobile/shared docs
└── AGENTS.md                    # this file
```

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| Backend run/config | `TicketPlatFormServer/TicketPlatFormServer/Program.cs` | DI, middleware, auth, SignalR, hosted services |
| Backend coding rules | `TicketPlatFormServer/AGENTS.md` | Backend root conventions |
| Backend repo constraints | `TicketPlatFormServer/TicketPlatFormServer/Repository/AGENTS.md` | Dapper/EF + connection safety |
| Backend service patterns | `TicketPlatFormServer/TicketPlatFormServer/Services/AGENTS.md` | AppException + business logic boundary |
| Mobile app conventions | `ticket_platform_mobile/AGENTS.md` | Riverpod, Freezed, generated files |
| Mobile core modules | `ticket_platform_mobile/lib/core/AGENTS.md` | network, router, theme, config |
| Mobile features | `ticket_platform_mobile/lib/features/AGENTS.md` | 16 features, Clean MVVM structure |
| Cross-team documentation | `docs/README.md` | Docs taxonomy and naming rules |

## CODE MAP
| Symbol | Type | Location | Refs | Role |
|--------|------|----------|------|------|
| `Program` (top-level) | Bootstrap | `TicketPlatFormServer/TicketPlatFormServer/Program.cs` | n/a | Backend composition root |
| `GlobalExceptionMiddleware` | Middleware | `TicketPlatFormServer/TicketPlatFormServer/Common/Exception/GlobalExceptionMiddleware.cs` | n/a | Unified API error envelope |
| `ChatHub` | SignalR hub | `TicketPlatFormServer/TicketPlatFormServer/Hubs/ChatHub.cs` | n/a | Realtime chat endpoint `/hubs/chat` |
| `AppConfig` | Config | `ticket_platform_mobile/lib/core/config/app_config.dart` | n/a | `--dart-define` driven runtime config |
| `ApiEndpoint` | Constants | `ticket_platform_mobile/lib/core/network/api_endpoint.dart` | n/a | All API URL constants |
| `goRouter` | Router | `ticket_platform_mobile/lib/core/router/app_router.dart` | n/a | GoRouter with all named routes |
| `ChatHub` | SignalR hub | `TicketPlatFormServer/TicketPlatFormServer/Hubs/ChatHub.cs` | n/a | Realtime chat endpoint |

## CONVENTIONS
- Backend keeps strict layer boundary: Controllers -> Services -> Repository -> DB.
- Mobile is feature-first (`lib/features/<feature>/{data,domain,presentation}`) plus shared core modules.
- `TicketPlatFormServer` is intentionally double-nested (`TicketPlatFormServer/TicketPlatFormServer`).
- Some root `.cs` files are utility references, not active app entrypoints.
- Mobile features list (16): auth, bank_account, chat, dispute, events, home, notification, payment, profile, reputation, sales_dashboard, search, sell, splash, ticketing, wishlist.
- Some root `.cs` files are utility references, not active app entrypoints.

## ANTI-PATTERNS (THIS PROJECT)
- Do not run backend repositories in parallel within one request (`Task.WhenAll` on repository calls is forbidden).
- Do not mix backend DTO and DB entity boundaries across layers.
- Do not edit generated Flutter files (`*.g.dart`, `*.freezed.dart`).
- Do not commit secrets (`.env`, API keys, service role keys).

## UNIQUE STYLES
- Backend docs and many XML comments are Korean; preserve language style in touched files.
- Backend uses hybrid EF Core + Dapper by query type, not one-size-fits-all ORM usage.
- Mobile uses script-driven `--dart-define` launch patterns in `ticket_platform_mobile/scripts/`.

## COMMANDS
```bash
# Backend
dotnet restore --project TicketPlatFormServer/TicketPlatFormServer.sln
dotnet build --project TicketPlatFormServer/TicketPlatFormServer.sln
dotnet run --project TicketPlatFormServer/TicketPlatFormServer/TicketPlatFormServer.csproj

# Backend DB restore
cd TicketPlatFormServer/TicketPlatFormServer/database_history && ./db_restore.sh

# Mobile
cd ticket_platform_mobile && flutter pub get
cd ticket_platform_mobile && flutter analyze
cd ticket_platform_mobile && ./scripts/run_dev.sh    # dev: APP_PROD=false
cd ticket_platform_mobile && ./scripts/run_prod.sh   # prod: APP_PROD=true
```

## NOTES
- No GitHub Actions workflow files currently exist; CI is manual/local.
- Backend test strategy is documented, but test project is not yet present in solution.
- Mobile `.env` required for local runs: copy `.env.example` → `.env`, set `KAKAO_NATIVE_APP_KEY`.
- `--dart-define` keys: `APP_PROD`, `API_BASE_URL_IOS`, `API_BASE_URL_ANDROID`, `TOSS_CLIENT_KEY`, `SUPABASE_URL`, `SUPABASE_ANON_KEY`.
- Backend test strategy is documented, but test project is not yet present in solution.
