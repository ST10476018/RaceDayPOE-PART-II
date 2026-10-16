
# RaceDay — Part 2 REST API

ASP.NET Core 8 Web API, C#, Entity Framework Core (Code First), SQL Server, server-side session authentication, Swagger/OpenAPI, xUnit tests, and GitHub Actions CI.

> **Important alignment note:** The screenshots describe Part 2 as the REST API. The 25 items you listed include MVC pages, dashboard views, Identity UI pages, and registration views (likely Part 3/tutorial material). Their video contents were not attached, so this project implements the API requirements shown in the screenshots and provides the backend endpoints those views can call. It does not pretend to reproduce unseen video code or claim to match an unavailable Part 1 ERD exactly. Compare `Database/RaceDaySchema.sql` with your approved Part 1 ERD/SQL before submission.

## Requirements
- Visual Studio 2022 or .NET 8 SDK
- SQL Server LocalDB, SQL Server Express, or SQL Server
- Internet access for first NuGet restore

## 1. Configure SQL Server

The default development connection string uses LocalDB:

`Server=(localdb)\MSSQLLocalDB;Database=RaceDayDb;Trusted_Connection=True;MultipleActiveResultSets=true;TrustServerCertificate=True`

If using SQL Server Express, change `ConnectionStrings:RaceDayDb` in `RaceDay.Api/appsettings.Development.json`, for example:

`Server=localhost\SQLEXPRESS;Database=RaceDayDb;Trusted_Connection=True;TrustServerCertificate=True;MultipleActiveResultSets=true`

If using SQL login, use `Server=YOUR_SERVER;Database=RaceDayDb;User Id=YOUR_USER;Password=YOUR_PASSWORD;TrustServerCertificate=True;MultipleActiveResultSets=true` and keep real credentials out of Git. Prefer user-secrets:

```bash
dotnet user-secrets init --project RaceDay.Api
dotnet user-secrets set "ConnectionStrings:RaceDayDb" "YOUR_CONNECTION_STRING" --project RaceDay.Api
```

## 2. Create database with EF Core migrations (recommended)

Open a terminal in the `RaceDay-Part2` folder:

```bash
dotnet tool install --global dotnet-ef --version 8.0.11
dotnet restore RaceDay.sln
dotnet ef migrations add InitialCreate --project RaceDay.Api --startup-project RaceDay.Api
dotnet ef database update --project RaceDay.Api --startup-project RaceDay.Api
```

If `dotnet-ef` is already installed, run `dotnet tool update --global dotnet-ef --version 8.0.11` instead of install. If your lecturer supplied a Part 1 SQL schema, update the entity classes to match it before creating the migration. Do not run the SQL schema script and migration against the same already-created database unless you know which tables already exist.

Alternative: open `Database/RaceDaySchema.sql` in SQL Server Management Studio and execute it against a new database. The SQL script is a schema reference; EF migrations are the normal Code First workflow.

## 3. Run the API

```bash
dotnet run --project RaceDay.Api
```

Open the Swagger page at `https://localhost:7142/swagger` (or the URL printed by `dotnet run`). Trust the local HTTPS development certificate if prompted:

```bash
dotnet dev-certs https --trust
```

## 4. Test session login in Swagger

1. `POST /api/auth/register` with JSON:

```json
{
  "fullName": "Sample Organizer",
  "email": "organizer@example.com",
  "password": "StrongPass1!",
  "role": "Organizer"
}
```

2. `POST /api/auth/login` with the same email/password. Keep the session cookie that the client receives. Swagger UI normally retains cookies for same-origin requests; if your browser does not, use a REST client that preserves cookies.
3. Use `GET /api/auth/me` to verify the session.
4. Create an event with `POST /api/events` as Organizer.
5. Create a category with `POST /api/categories`, then assign it with `POST /api/events/{eventId}/categories/{categoryId}`.
6. Register and log in as a Participant in a separate browser/client session, then enrol using `POST /api/enrolments/{eventId}` with `{"categoryId": 1}`.
7. As Organizer, list enrolments using `GET /api/enrolments/event/{eventId}` and record a result using `POST /api/results/events/{eventId}`.

Example event JSON:

```json
{
  "name": "Durban 10K",
  "description": "Community running event",
  "eventDate": "2026-11-15T07:00:00Z",
  "location": "Durban",
  "distanceKm": 10,
  "eventType": "run"
}
```

Example result JSON:

```json
{
  "participantId": 2,
  "finishTime": "00:42:15",
  "finishPosition": 1
}
```

## API endpoint summary

| Method | Endpoint | Access | Purpose |
|---|---|---|---|
| POST | `/api/auth/register` | Public | Register Organizer or Participant |
| POST | `/api/auth/login` | Public | Verify credentials and start session |
| POST | `/api/auth/logout` | Session | Clear session |
| GET | `/api/auth/me` | Session | Current user/session details |
| GET/PUT | `/api/users/me` | Session | View/update own profile |
| GET | `/api/events` | Public | List events |
| GET | `/api/events/{id}` | Public | View event and its categories |
| POST/PUT/DELETE | `/api/events` and `/api/events/{id}` | Organizer, owner only | Manage own events |
| POST | `/api/events/{id}/categories/{categoryId}` | Organizer, owner only | Assign category to event |
| GET | `/api/categories` | Public | List categories |
| POST/PUT | `/api/categories` and `/api/categories/{id}` | Organizer | Create/update categories |
| POST | `/api/enrolments/{eventId}` | Participant | Enrol in an event/category |
| GET | `/api/enrolments/mine` | Participant | View own enrolments |
| GET | `/api/enrolments/event/{eventId}` | Organizer, owner only | View enrolments for own event |
| POST | `/api/results/events/{eventId}` | Organizer, owner only | Add/update result for an enrolled participant |
| GET | `/api/results/mine` | Participant | View own results |
| GET | `/api/results/events/{eventId}` | Authenticated | View results; organisers only for their own events |
| GET | `/api/health` | Public | Health check |

## Security notes
- Passwords are hashed with ASP.NET Core's `PasswordHasher<TUser>`; plaintext passwords are never stored.
- The session stores user ID, role, and display name server-side. The session cookie is HttpOnly and has a 30-minute idle timeout.
- API role checks are explicit. An Organizer can only modify events they own; Participants cannot call Organizer operations.
- For production, use HTTPS only, configure a persistent/distributed session store, add CSRF protection for browser-cookie workflows, apply rate limiting, add email verification/password reset, and move secrets to a secret manager.
- For a real deployment, consider ASP.NET Core Identity and its role tables if the final Part 1/Part 3 design explicitly requires Identity. This assignment's screenshot calls for session management, so this API uses a simple server-side session implementation.

## 5. Run unit tests

```bash
dotnet test RaceDay.sln --configuration Release
```

Tests cover accepted/rejected role names, password hashing/verification, and supported event types. These are unit tests for core validation/security helpers; add endpoint integration tests against a test host and isolated database if required by your lecturer's marking rubric.

## 6. GitHub repository and CI/CD

1. Create a GitHub repository and push this project to a branch named `part-2-api` or `main`.
2. The workflow at `.github/workflows/dotnet.yml` restores, builds, and tests the solution on every push and pull request.
3. Open the **Actions** tab and capture a screenshot of a green successful run. Put the real screenshot in this README before submission; do not use a fabricated screenshot.
4. The brief requires at least 20 meaningful commits. Make small real commits as you develop/test the features; do not create empty or misleading commits just to meet the count.
5. Record your unlisted YouTube demonstration URL here after you make and upload your own demonstration:

`Demo video: ADD_YOUR_UNLISTED_YOUTUBE_URL_HERE`

## Suggested demonstration order
1. Explain the folders and the EF Core model/schema.
2. Show migrations/database connection and Swagger.
3. Register/log in, then show the session-protected `/api/auth/me` endpoint.
4. Demonstrate Organizer-only event/category management and ownership restriction.
5. Demonstrate Participant enrolment and Organizer enrolment listing.
6. Record/view results.
7. Run `dotnet test` and show the successful GitHub Actions workflow.
