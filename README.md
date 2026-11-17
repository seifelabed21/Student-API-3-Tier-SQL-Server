# Student API 3-Tier with SQL Server

> A personal learning project by **Seif Elabed** while studying RESTful APIs at [Programming Advices Academy](https://programmingadvices.com/courses/enrolled/2537475) (Arabic).

I built this project in Visual Studio while learning how to move from a simple in-memory API to a real **.NET 8 Web API** backed by **SQL Server**. I tested the endpoints with Swagger and managed the database with SQL Server Management Studio 2022.

This repository is both a working student-management API and a snapshot of an important stage in my learning journey: introducing a three-tier architecture, DTOs, ADO.NET, and stored procedures for the first time.

## Project at a glance

| Area | Details |
| --- | --- |
| API | ASP.NET Core Web API on .NET 8 |
| Database | SQL Server database named `StudentsDB` |
| Architecture | Presentation/API, Business Logic, and Data Access layers |
| Database access | ADO.NET through `Microsoft.Data.SqlClient` |
| Database operations | Stored procedures |
| API testing | Swagger / OpenAPI |
| Additional client | C# console application using `HttpClient` |
| Development tools | Visual Studio 2022 and SSMS 2022 |

## What I learned

This project helped me practice:

- Designing REST-style endpoints with HTTP verbs.
- Returning meaningful status codes such as `200`, `201`, `400`, and `404`.
- Separating controller, business, and data-access responsibilities.
- Passing `StudentDTO` objects between layers instead of exposing internal objects.
- Calling SQL Server stored procedures with ADO.NET.
- Reading multiple rows with `SqlDataReader`.
- Reading one scalar value with `ExecuteScalar`.
- Using an output parameter to receive the ID of a newly inserted student.
- Using a `Find` and `Save` pattern for updating an existing student.
- Consuming the API from a separate console client.

## Three-tier architecture

```text
Client (Swagger, PowerShell, or console client)
                    |
                    v
Presentation / API layer
MyFirstRestAPI_Porject
StudentAPIController
                    |
                    v
Business Logic layer
StudentAPIBusinessLayer
Student class, Find(), Save(), and delegation
                    |
                    v
Data Access layer
StudentDataAccessLayer
StudentData, DTOs, ADO.NET, and stored procedures
                    |
                    v
SQL Server
StudentsDB
```

Each layer knows only about the layer immediately below it:

- **API layer:** receives HTTP requests, validates basic input, calls the business layer, and returns HTTP responses.
- **Business layer:** represents student operations and controls the `AddNew`/`Update` mode used by `Save()`.
- **Data access layer:** opens SQL connections, executes stored procedures, maps database rows to DTOs, and returns results.
- **Database:** stores the data and performs database-side operations such as filtering passed students.

The API does not access SQL Server directly. A request flows through:

```text
HTTP request
  -> StudentsController
  -> Student business class
  -> StudentData
  -> stored procedure
  -> SQL Server
  -> DTO
  -> JSON response
```

## Database design

The restored database contains one main table:

```sql
CREATE TABLE Students (
    Id INT IDENTITY(1,1) PRIMARY KEY,
    Name NVARCHAR(100) NOT NULL,
    Age INT NOT NULL,
    Grade INT NOT NULL
);
```

The API uses these stored procedures:

| Stored procedure | Responsibility | ADO.NET approach |
| --- | --- | --- |
| `SP_GetAllStudents` | Return every student | `ExecuteReader` |
| `SP_GetPassedStudents` | Return students where `Grade >= 50` | `ExecuteReader` |
| `SP_GetAverageGrade` | Return the average grade | `ExecuteScalar` |
| `SP_GetStudentById` | Return one student by ID | Parameterized `ExecuteReader` |
| `SP_AddStudent` | Insert a student and return its new ID | Output parameter |
| `SP_UpdateStudent` | Update an existing student | `ExecuteNonQuery` |
| `SP_DeleteStudent` | Delete a student and report whether it was deleted | Affected-row result |

The passing-student filter is performed in SQL Server through `SP_GetPassedStudents`, rather than loading every student and filtering in C#.

## Repository structure

```text
Student API Project With DB/
├── 01 StudentAPIServer with DB/
│   ├── StudentApi.sln
│   ├── MyFirstRestAPI_Porject/
│   │   ├── Controllers/StudentAPIController.cs
│   │   ├── Program.cs
│   │   ├── StudentApi.csproj
│   │   └── Properties/launchSettings.json
│   ├── StudentAPIBusinessLayer/
│   │   ├── Student.cs
│   │   └── StudentAPIBusinessLayer.csproj
│   └── StudentDataAccessLayer/
│       ├── StudentData.cs
│       └── StudentDataAccessLayer.csproj
├── 02 StudentApiClient with DB/
│   ├── StudentApiClient.sln
│   └── StudentApiClient/
│       ├── Program.cs
│       └── StudentApiClient.csproj
├── Database/
│   └── StudentsDB.bak
├── project specs.pdf
└── README.md
```

The `DataSimulation` and `Model` files in the server are remnants of the earlier in-memory version. The current database-backed implementation uses `StudentDataAccessLayer` and `StudentDTO`.

## API endpoints

The controller route is `api/Students`.

| Method | Route | Success behavior | Main error behavior |
| --- | --- | --- | --- |
| `GET` | `/api/Students/All` | `200` with all students | `404` when no students exist |
| `GET` | `/api/Students/Passed` | `200` with students where grade is at least 50 | `404` when no passing students exist |
| `GET` | `/api/Students/AverageGrade` | `200` with one numeric average | Returns `0` when the database has no grades |
| `GET` | `/api/Students/{id}` | `200` with one `StudentDTO` | `400` for IDs below 1; `404` if not found |
| `POST` | `/api/Students` | `201` with the created student and a `Location` route | `400` for invalid data |
| `PUT` | `/api/Students/{id}` | `200` with the updated `StudentDTO` | `400` for invalid data; `404` if not found |
| `DELETE` | `/api/Students/{id}` | `200` with a deletion message | `400` for IDs below 1; `404` if not found |

The expected student JSON shape is:

```json
{
  "id": 0,
  "name": "Mazen Abdullah",
  "age": 20,
  "grade": 85
}
```

For `POST` and `PUT`, `id` can be omitted or left as `0`; the database generates the ID for a new student.

## Prerequisites

Install:

1. Visual Studio 2022 with the **ASP.NET and web development** workload, or the .NET 8 SDK.
2. SQL Server, either a default local instance or SQL Server Express.
3. SQL Server Management Studio 2022.
4. Git, if cloning the repository.

Check the .NET SDK:

```powershell
dotnet --version
```

## Setup the database with SSMS 2022

The API expects a database named `StudentsDB` containing the `Students` table and the stored procedures listed above. The repository includes `Database\StudentsDB.bak`.

1. Start the SQL Server service and note the instance name. Common examples are `localhost` and `.\SQLEXPRESS`.
2. Open SQL Server Management Studio 2022.
3. Connect to the target Database Engine instance.
4. Right-click **Databases** and select **Restore Database...**.
5. Select **Device**, browse to `Database\StudentsDB.bak`, and add the backup file.
6. Select the backup set and set the destination database name to `StudentsDB`.
7. On **Files**, confirm that the MDF and LDF destination folders exist on the SQL Server machine.
8. On **Options**, select **Overwrite the existing database (WITH REPLACE)** only when intentionally replacing an existing learning database.
9. Select **OK**, refresh **Databases**, and confirm that `StudentsDB` was restored.
10. Expand `StudentsDB > Programmability > Stored Procedures` and verify the required procedures.

### Configure the SQL Server connection

The original course implementation keeps its connection string in:

`01 StudentAPIServer with DB\StudentDataAccessLayer\StudentData.cs`

It currently expects a local SQL login similar to:

```text
Server=localhost;Database=StudentsDB;User Id=sa;Password=YOUR_PASSWORD;Encrypt=False;TrustServerCertificate=True;Connection Timeout=30;
```

If your instance or password is different, update the connection string before running the API. For SQL Server Express, the server value may be:

```text
Server=.\SQLEXPRESS;
```

> **Security warning:** the source uses `YOUR_PASSWORD` as a placeholder so the repository does not contain a real credential. Replace it for local testing and, before publishing a deployment, move the connection string to user secrets, environment variables, or a deployment secret store. Never commit a real production password.

## Run the API in Visual Studio

1. Open `01 StudentAPIServer with DB\StudentApi.sln`.
2. Select `MyFirstRestAPI_Porject` as the startup project if necessary.
3. Choose the `http` or `https` launch profile.
4. Press **F5** or select **Debug > Start Without Debugging**.
5. Open Swagger:

   - HTTP: `http://localhost:5215/swagger`
   - HTTPS: `https://localhost:7217/swagger`

Swagger is enabled by the application when `ASPNETCORE_ENVIRONMENT` is `Development`, which is set by the included launch profiles.

## Run the API from the command line

```powershell
cd "01 StudentAPIServer with DB\MyFirstRestAPI_Porject"
dotnet restore
dotnet build
dotnet run --launch-profile http
```

Open `http://localhost:5215/swagger`. To use HTTPS instead:

```powershell
dotnet run --launch-profile https
```

If the HTTPS development certificate is not trusted, use the HTTP profile for local testing.

## Test and use the API

Swagger is the recommended testing tool for this project:

1. Start the API.
2. Open `/swagger`.
3. Expand an endpoint.
4. Select **Try it out**.
5. Enter any required ID or JSON body.
6. Select **Execute** and inspect the response status and body.

The HTTP base URL used by the included launch profile is:

```text
http://localhost:5215/api/Students
```

Example PowerShell requests:

```powershell
$baseUrl = "http://localhost:5215/api/Students"

Invoke-RestMethod "$baseUrl/All"
Invoke-RestMethod "$baseUrl/Passed"
Invoke-RestMethod "$baseUrl/AverageGrade"
Invoke-RestMethod "$baseUrl/1"

$student = @{
    name = "Mazen Abdullah"
    age = 20
    grade = 85
} | ConvertTo-Json

Invoke-RestMethod $baseUrl -Method Post -ContentType "application/json" -Body $student
Invoke-RestMethod "$baseUrl/1" -Method Put -ContentType "application/json" -Body $student
Invoke-RestMethod "$baseUrl/1" -Method Delete
```

Use safe test IDs and data because these requests modify the restored database.

## Run the console client

The second solution demonstrates consuming the API with `HttpClient`.

1. Start the API first.
2. Open `02 StudentApiClient with DB\StudentApiClient.sln`, or use the command line.
3. Confirm that `02 StudentApiClient with DB\StudentApiClient\Program.cs` points to the running API:

```csharp
httpClient.BaseAddress = new Uri("http://localhost:5215/api/Students/");
```

4. Run it:

```powershell
cd "02 StudentApiClient with DB\StudentApiClient"
dotnet restore
dotnet run
```

The sample client calls every API operation. It reads students, requests passed students and the average grade, checks IDs 1 and 20, adds a sample student, deletes ID 1, and updates ID 2. Review those IDs and sample values before running it against a database you want to preserve.

## Deployment

This is a straightforward Windows deployment of the published .NET API against an existing SQL Server instance.

### 1. Prepare SQL Server

Restore `Database\StudentsDB.bak` on the target SQL Server and verify the `Students` table and all required stored procedures. Give the deployment account only the permissions needed to use `StudentsDB` and execute the procedures.

### 2. Configure secrets and connectivity

Set the target SQL Server host, authentication, database name, and credentials. Do not deploy the sample `sa` password or keep production credentials in source code. Confirm that the API machine can reach SQL Server through the configured port and firewall.

### 3. Publish the API

From the API project directory:

```powershell
cd "01 StudentAPIServer with DB\MyFirstRestAPI_Porject"
dotnet restore
dotnet publish -c Release -o .\publish
```

Copy the generated `publish` folder to the target machine. The published output, not the development `bin` or `obj` folders, is the deployment artifact.

### 4. Run or host the published API

For a direct process test:

```powershell
cd C:\Path\To\publish
.\StudentApi.exe
```

For a server deployment, host the published application behind IIS or another reverse proxy, configure HTTPS, and run it as a managed application process or Windows service. Configure the environment and listening URL for the host.

The current application enables Swagger only in the Development environment. Keep it disabled or protected in a public production deployment.

### 5. Verify the deployment

1. Start the published API.
2. Call `GET /api/Students/All`.
3. Test one read request and one safe write request.
4. Confirm that the API can connect to `StudentsDB`.
5. Check application and SQL Server logs if a request fails.

## Current limitations and honest notes

- The API uses synchronous data-access methods and a static connection string, reflecting the original course implementation.
- Basic validation checks names and non-negative age and grade values; stronger validation is still needed.
- There is no authentication or authorization. Do not expose this API directly to the public internet.
- The project has no automated test suite.
- The included backup contains sample data; back it up before experimenting.
- `DataSimulation\StudentDataSimulation.cs` and `Model\Student.cs` are commented-out remnants from the earlier in-memory version.
- The written course specification describes `DELETE` as checking affected rows with `ExecuteNonQuery`; the current source should be treated as the final behavior when those details differ.

## Future improvements

- Move the connection string into configuration and secret management.
- Add authentication and authorization.
- Use asynchronous database and controller methods.
- Add unit and integration tests.
- Add structured logging and centralized exception handling.
- Add a repeatable SQL initialization script or migrations.
- Add pagination and stronger request validation.
- Add CI/CD and production hosting configuration.

## Author

**Seif Elabed**

This project was created while studying RESTful APIs through Programming Advices Academy. I am keeping it as a personal record of learning ASP.NET Core, SQL Server, ADO.NET, stored procedures, layered architecture, and API client development.
