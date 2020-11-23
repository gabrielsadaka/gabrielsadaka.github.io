# Deploying a .NET Core API to GCP Cloud Run with Cloud SQL Postgres using Pulumi and Batect

## Motivation

Having never worked with GCP or [Pulumi](https://www.pulumi.com/), I thought it would be fun to explore the technologies by building a .NET Core API following as many best practices as possible for production ready cloud native services. I also wanted to explore more about [Batect](https://batect.dev/), which is a tool that I have used in past projects but have never utilized in a greenfield project. The intention was to develop an API that implemented every best practice I could think of for developing cloud native services and then share what I learnt throughout the process. Rather than wait until I have completed the journey I have decided to share my learnings so far.

## What is it?

The API is a very simple .NET Core Web API that creates and retrieves weather forecasts. The forecasts are stored in a GCP Cloud SQL Postgres database while the API is deployed as a Docker container running on GCP Cloud Run. It has tests written at every level of the test pyramid relevant to the component, starting at unit tests for the business logic, controllers and repository layer, leading to integration tests that run against a Postgres docker container.

GCP Secret Manager is used to store the database password which is accessed by the .NET Core API and the database schema migration tool. The database schema migrations are applied using [Flyway](https://flywaydb.org/) running in a docker container, defined as a task in Batect. They are applied during deployment and every time the integration tests are run to ensure a clean database is available for each test run.

The source code is hosted on GitHub using GitHub actions to automatically build and test the application then deploy it and it's infrastructure to GCP using Pulumi. It uses Batect to define the tasks that are run during the CI/CD pipeline to enable them to run consistently during local development and in the GitHub actions.

There are a few tools used for static analysis and for security/dependency scanning to ensure the application has no known vulnerabilities. The ASP.NET Core API uses [SonarAnalyzer.CSharp](https://www.nuget.org/packages/SonarAnalyzer.CSharp/) and [Microsoft.CodeAnalysis.FxCopAnalyzers](https://www.nuget.org/packages/Microsoft.CodeAnalysis.FxCopAnalyzers/) both of which are static analysis libraries for C# that detect code patterns that introduce security vulnerabilities or other flaws such as memory leaks. [GitHub CodeQL](https://securitylab.github.com/tools/codeql) is another static analysis tool that is used as a GitHub action. [Dependabot](https://dependabot.com/) is used for scanning the .NET dependencies to ensure they are up to date. [Trivy](https://github.com/aquasecurity/trivy) is used to scan the .NET Core Docker image for known vulnerabilities while [Dockle](https://github.com/goodwithtech/dockle) is a Dockerfile linter for detecting security flaws.

The principle of least privilege has been followed to ensure that the account that deploys the infrastructure and the application only has the privileges it requires to perform those tasks.

## What does it look like?

### The Build Pipeline

Although the code is hosted on GitHub and uses GitHub actions to automate the building and deployment, it makes use of very little GitHub actions specific functionality. Instead it uses Batect to what happens at each step, enabling the developer to validate that it will all work locally before pushing the code and waiting to see if it will succeed. It also ensures that the tasks can be run consistently on any machine with very little setup required removing the concern that arises from "doesn't run on my machine".

To build the application the standard .NET Core SDK base image is defined as a container in Batect, along with the volume to map the code, the nuget cache and the `obj` folder to optimise the performance of subsequent builds by caching intermediate artefacts. A task is also defined that will use that container to build the API.

```yaml
containers:
---
build-env:
  image: mcr.microsoft.com/dotnet/core/sdk:3.1
  volumes:
    - local: .
      container: /code
      options: cached
    - type: cache
      name: nuget-cache
      container: /root/.nuget/packages
    - type: cache
      name: weatherApi-obj
      container: /code/src/WeatherApi/obj
    - type: cache
      name: weatherApi-tests-obj
      container: /code/src/WeatherApi.Tests/obj
  working_directory: /code
---
tasks:
---
build:
  description: Build Weather API
  run:
    container: build-env
    command: dotnet build
```

To run the integration tests the same `build-env` Docker container is used along with a Docker container for Postgres and a Docker container for Flyway which will run the schema migrations. A task is defined that will run the tests that depends on the Flyway migrations completing which in turn depends on Postgres running. The integration tests can be run locally the exact same way they are run in the build pipeline to enable the developer to be confident that the changes they have made won't result in a broken build.

```yaml
containers:
---
postgres:
  image: postgres:13
  ports:
    - local: 5432
      container: 5432
  environment:
    POSTGRES_DB: weather_db
    POSTGRES_USER: weather_user
    POSTGRES_PASSWORD: $POSTGRES_PASSWORD
flyway-migrator:
  build_directory: db
  volumes:
    - local: db/migrations
      container: /migrations
      options: cached
    - local: db/scripts
      container: /scripts
      options: cached
  environment:
    DB_PORT: "5432"
    DB_SCHEMA_NAME: public
    DB_MIGRATIONS_LOCATION: filesystem:/migrations
---
tasks:
---
migrate-integration-test-db:
  description: Run database migrations for integration tests
  dependencies:
    - postgres
  run:
    container: flyway-migrator
    entrypoint: /scripts/run-migrations.sh
    environment:
      DB_HOST: postgres
      DB_USERNAME: weather_user
      DB_PASSWORD: $POSTGRES_PASSWORD
      DB_NAME: weather_db

test:
  description: Test Weather API
  run:
    container: build-env
    command: dotnet test
    environment:
      ASPNETCORE_ENVIRONMENT: test
      WEATHERDB__PASSWORD: $POSTGRES_PASSWORD
  dependencies:
    - postgres
  prerequisites:
    - migrate-integration-test-db
```

The GitHub action now only needs to run the Batect task called `test` which will trigger Postgres to start, the Flyway migrations to run and then the API unit and integration tests to run. It uses the default Ubuntu image, first checking out the code then calling Batect to run the task.

```yaml
jobs:
---
build-test:
  runs-on: ubuntu-latest
  steps:
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Build and test application
      run: ./batect test
      env:
        POSTGRES_PASSWORD: ${{ secrets.POSTGRES_PASSWORD }}
---

```

### The Infrastructure

Deploying the infrastructure is performed in two steps by two different GCP accounts. There is an account that has the permissions to make IAM changes and enable GCP APIs named, `iam-svc`, there are instructions in the readme of the repository to create this account. The IAM account then creates a CI account named, `ci-svc`, with only the permissions it requires to deploy the infrastructure and deploy the application to GCP Cloud Run, along with an account named, `weather-api-cloud-run`, that is the service account the Cloud Run app runs as.

The deployment of the IAM changes, infrastructure and the app itself are all performed by Pulumi. The tasks performed in the GitHub Actions like the build and test are all defined as containers/tasks in Batect.

```yaml
containers:
---
pulumi:
  image: pulumi/pulumi:v2.12.1
  working_directory: /app
  volumes:
    - local: infra
      container: /app
      options: cached
    - local: /var/run/docker.sock
      container: /var/run/docker.sock
  environment:
    APP_NAME: $APP_NAME
    GOOGLE_CREDENTIALS: $GOOGLE_CREDENTIALS
    GOOGLE_SERVICE_ACCOUNT: $GOOGLE_SERVICE_ACCOUNT
    GOOGLE_PROJECT: $GOOGLE_PROJECT
    GOOGLE_REGION: $GOOGLE_REGION
    PULUMI_ACCESS_TOKEN: $PULUMI_ACCESS_TOKEN
    GITHUB_SHA: $GITHUB_SHA
  entrypoint: bash

tasks:
---
deploy-iam:
  description: Deploy IAM using Pulumi
  run:
    container: pulumi
    command: deploy-iam.sh
deploy-infra:
  description: Deploy infra using Pulumi
  run:
    container: pulumi
    command: deploy-infra.sh
    environment:
      DB_NAME: $DB_NAME
      DB_USERNAME: $DB_USERNAME
```

The two GCP accounts, the IAM changes and the enabling of the GCP APIs are declared using the GCP Pulumi library, defined in the `deploy-iam` task. Similar to Terraform, Pulumi maintains the latest state of the deployment, comparing that to the declared state in the latest commit, applying only the changes that were made.

```typescript
...
const ciServiceAccount = new gcp.serviceaccount.Account(`ci-svc`, {
    accountId: `ci-svc`,
    description: `CI Service Account`,
    displayName: `CI Service Account`
}, {dependsOn: enableGcpApis.enableIamApi});

const storageAdminIamBinding = new gcp.projects.IAMBinding(`ci-svc-storage-admin`, {
    members: [ciServiceAccountEmail],
    role: "roles/storage.admin"
}, {parent: ciServiceAccount, dependsOn: enableGcpApis.enableIamApi});

...

export const enableCloudRunApi= new gcp.projects.Service("EnableCloudRunApi", {
    service: "run.googleapis.com",
});
```

The only infrastructure deployed for this application is the container registry that stores the build Docker images that GCP Cloud Run depends on and the GCP Cloud SQL Postgres instance that the application writes and reads weather forecasts from. They are both defined using Pulumi, defined in the `deploy-infra` task.

```typescript
const registry = new gcp.container.Registry("weather-registry");

...

export const databaseInstance = new gcp.sql.DatabaseInstance(`${config.appName}-db`, {
    name: `${config.appName}-db`,
    databaseVersion: "POSTGRES_12",
    settings: {
        tier: "db-f1-micro",
        ipConfiguration: {
            ipv4Enabled: true,
            requireSsl: true
        }
    },
});

export const database = new gcp.sql.Database(`${config.appName}-db`, {
    name: config.dbName,
    instance: databaseInstance.id
});
```

### The Database

The database schema migrations are performed by Flyway running in Docker container, defined as a Batect container/task. The task first calls GCP Secrets Manager to retrieve the database password then connects to the Cloud SQL Postgres instance using the Cloud SQL Proxy and then performs the schema migrations.

```yaml
containers:
---
flyway-migrator:
  build_directory: db
  volumes:
    - local: db/migrations
      container: /migrations
      options: cached
    - local: db/scripts
      container: /scripts
      options: cached
  environment:
    DB_PORT: "5432"
    DB_SCHEMA_NAME: public
    DB_MIGRATIONS_LOCATION: filesystem:/migrations
---
tasks:
---
migrate-db:
  description: Run database migrations
  run:
    container: flyway-migrator
    entrypoint: /scripts/run-migrations-gcloud.sh
    environment:
      GOOGLE_PROJECT: $GOOGLE_PROJECT
      GOOGLE_REGION: $GOOGLE_REGION
      GOOGLE_CREDENTIALS: $GOOGLE_CREDENTIALS
      DB_INSTANCE: $DB_INSTANCE
      DB_HOST: localhost
      DB_NAME: $DB_NAME
      DB_USERNAME: $DB_USERNAME
      DB_PASSWORD_SECRET_ID: $DB_PASSWORD_SECRET_ID
      DB_PASSWORD_SECRET_VERSION: $DB_PASSWORD_SECRET_VERSION
```

### The Deployment (Dockerfile/Batect/GitHub Actions)

#### Build, scan and publish image

To run the application in GCP Cloud Run a Docker image needs to be built and deployed to a container registry that is accessible by the service. I chose GCP Container Registry since it was the easiest to get working with Cloud Run. Just like the previous tasks the build, scan and publish image step is defined as a Batect container/task.

```yaml
containers:
---
gcloud-sdk:
  image: google/cloud-sdk:314.0.0-alpine
  working_directory: /app
  volumes:
    - local: .
      container: /app
      options: cached
    - local: /var/run/docker.sock
      container: /var/run/docker.sock
  environment:
    APP_NAME: $APP_NAME
    GOOGLE_CREDENTIALS: $GOOGLE_CREDENTIALS
    GOOGLE_PROJECT: $GOOGLE_PROJECT
    GITHUB_SHA: $GITHUB_SHA
---
tasks:
---
build-scan-image:
  description: Build and scan docker image for vulnerabilities
  run:
    container: gcloud-sdk
    command: infra/build-scan-image.sh
push-image:
  description: Push docker image to GCR
  run:
    container: gcloud-sdk
    command: infra/push-image.sh
```

After the image is built, it is scanned for vulnerabilities using Trivy and Dockle to ensure that there are no dependencies with known vulnerabilities and that the Dockerfile follows all of the best security practices. One of the vulnerabilities Dockle identified for me was that I forgot to change the running user at the end of the Dockerfile so that the application didn't run as the root user.

```bash
IMAGE_TAG=gcr.io/"$GOOGLE_PROJECT"/"$APP_NAME":"$GITHUB_SHA"

docker build . -t "$IMAGE_TAG"

docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:0.12.0 \
    --exit-code 1 --no-progress --severity CRITICAL "$IMAGE_TAG"

docker run --rm  --rm -v /var/run/docker.sock:/var/run/docker.sock -i goodwithtech/dockle:v0.3.1 \
     --exit-code 1 --exit-level warn "$IMAGE_TAG"
```

```Dockerfile
---
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1 AS runtime
WORKDIR /app

COPY --from=build /app/out ./

RUN useradd -m -s /bin/bash dotnet-user
USER dotnet-user

ENV ASPNETCORE_URLS=http://*:8080

ENTRYPOINT ["dotnet", "WeatherApi.dll"]
```

#### Deploy app to GCP Cloud Run

Once the API is built and published to the Container Registry it is then deployed to GCP Cloud Run using Pulumi, which is again defined as a Batect container/task. It uses the same Pulumi container definition declared above for the IAM and infrastructure tasks.

```yaml
tasks:
---
deploy:
  description: Deploy Weather API to GCP Cloud Run using Pulumi
  run:
    container: pulumi
    command: deploy-app.sh
    environment:
      GOOGLE_RUN_SERVICE_ACCOUNT: $GOOGLE_RUN_SERVICE_ACCOUNT
      DB_INSTANCE: $DB_INSTANCE
      ENVIRONMENT: $ENVIRONMENT
```

The GCP Cloud Run service is defined to use the container port 8080, as the default port of 80 cannot be bound to be unprivileged users. It is also defined with a dependency on the GCP Cloud SQL Postgres instance, enabling it access it via the Cloud SQL Proxy. It is set to be visible to all users and is therefore public by default, this can be changed to only allow authenticated GCP users if needed.

```typescript
const weatherApi = new gcp.cloudrun.Service(appName, {
  location,
  name: appName,
  template: {
    spec: {
      containers: [
        {
          image: `gcr.io/${gcp.config.project}/${appName}:${gitSha}`,
          envs: [
            {
              name: "GOOGLE_PROJECT",
              value: gcp.config.project,
            },
            {
              name: "ASPNETCORE_ENVIRONMENT",
              value: environment,
            },
          ],
          ports: [
            {
              containerPort: 8080,
            },
          ],
        },
      ],
      serviceAccountName: googleCloudRunServiceAccount,
    },
    metadata: {
      annotations: {
        "autoscaling.knative.dev/maxScale": "2",
        "run.googleapis.com/cloudsql-instances": cloudSqlInstance,
      },
    },
  },
});

// Open the service to public unrestricted access
const iamWeatherApi = new gcp.cloudrun.IamMember(`${appName}-everyone`, {
  service: weatherApi.name,
  location,
  role: "roles/run.invoker",
  member: "allUsers",
});
```

### The API

The API consists of a single controller that exposes two simple endpoints, one to create weather forecasts and another to retrieve those forecasts. It uses [MediatR](https://github.com/jbogard/MediatR) to implement a CQRS style architecture where each query and command has separate classes to represent the data transfer objects and the handlers.

```csharp
[HttpGet]
public async Task<IActionResult> Get([FromQuery] GetWeatherForecastQuery query, CancellationToken ct = default)
{
    var response = await _mediator.Send(query, ct);
    return Ok(response);
}

[HttpPost]
public async Task<IActionResult> Post(AddWeatherForecastCommand command, CancellationToken ct = default)
{
    var response = await _mediator.Send(command, ct);
    return Ok(response);
}
```

The add weather forecast command handler creates a new instance of the `WeatherForecast` entity and calls the repository to store in the database.

```csharp
public async Task<AddWeatherForecastResponse> Handle(AddWeatherForecastCommand request,
    CancellationToken cancellationToken)
{
    var id = Guid.NewGuid();

    await _weatherForecastsRepository.AddWeatherForecast(new WeatherForecast(id,
        request.City, request.ForecastDate, request.Forecast), cancellationToken);

    return new AddWeatherForecastResponse(id);
}
```

While the get weather forecast query handler retrieves the forecast from the database by calling the repository, calling a `NotFoundException` when it doesn't exist in the database.

```csharp
public async Task<GetWeatherForecastResponse> Handle(GetWeatherForecastQuery request,
    CancellationToken cancellationToken)
{
    var weatherForecast = await _weatherForecastsRepository.GetWeatherForecast(request.City, request.ForecastDate, cancellationToken);

    if (weatherForecast == null) throw new NotFoundException();

    return new GetWeatherForecastResponse(weatherForecast.Id, weatherForecast.City, weatherForecast.ForecastDate.UtcDateTime, weatherForecast.Forecast);
}
```

The `NotFoundException` is mapped to a [problem details](https://tools.ietf.org/html/rfc7807) HTTP 404 error using the .NET library [Hellang.Middleware.ProblemDetails](https://www.nuget.org/packages/Hellang.Middleware.ProblemDetails/).

```csharp
services.AddProblemDetails(opts =>
{
    var showExceptionDetails = Configuration["Settings:ShowExceptionDetails"].Equals("true", StringComparison.InvariantCultureIgnoreCase);
    opts.ShouldLogUnhandledException = (ctx, ex, pb) => showExceptionDetails;
    opts.IncludeExceptionDetails = (ctx, ex) => showExceptionDetails;
    opts.MapToStatusCode<NotFoundException>(StatusCodes.Status404NotFound);
});
```

Entity Framework Core is used for data access within the API to read and write data to the Postgres database. While running locally and in integration tests it uses a Postgres Docker container to host the database, while using GCP Cloud SQL when it is running in GCP Cloud Run. GCP Secrets Manager is used to store the database password and is therefore retrieved when the application starts if it is running in Cloud Run.

```csharp
private static void ConfigureDbContext(IServiceCollection services, IConfiguration configuration)
{
    var dbSettings = configuration.GetSection("WeatherDb");
    var dbSocketDir = dbSettings["SocketPath"];
    var instanceConnectionName = dbSettings["InstanceConnectionName"];
    var databasePasswordSecret = GetDatabasePasswordSecret(dbSettings);
    var connectionString = new NpgsqlConnectionStringBuilder
    {
        Host = !string.IsNullOrEmpty(dbSocketDir)
            ? $"{dbSocketDir}/{instanceConnectionName}"
            : dbSettings["Host"],
        Username = dbSettings["User"],
        Password = databasePasswordSecret,
        Database = dbSettings["Name"],
        SslMode = SslMode.Disable,
        Pooling = true
    };

    services.AddDbContext<WeatherContext>(options =>
        options
            .UseNpgsql(connectionString.ToString())
            .UseSnakeCaseNamingConvention());
}

private static string GetDatabasePasswordSecret(IConfiguration dbSettings)
{
    var googleProject = Environment.GetEnvironmentVariable("GOOGLE_PROJECT");

    if (string.IsNullOrEmpty(googleProject)) return dbSettings["Password"];

    var dbPasswordSecretId = dbSettings["PasswordSecretId"];
    var dbPasswordSecretVersion = dbSettings["PasswordSecretVersion"];

    var client = SecretManagerServiceClient.Create();

    var secretVersionName = new SecretVersionName(googleProject, dbPasswordSecretId, dbPasswordSecretVersion);

    var result = client.AccessSecretVersion(secretVersionName);

    return result.Payload.Data.ToStringUtf8();
}
```

To test the API all application code is covered by unit tests, with higher level integration tests designed to ensure the API functions correctly when it is running. The integration tests utilise the ASP.NET testing library to run the API in an in memory server communicating with a Postgres Docker container to store and retrieve the weather forecasts.

```csharp
protected override void ConfigureWebHost(IWebHostBuilder builder)
{
    if (builder == null) throw new ArgumentNullException(nameof(builder));
    builder.ConfigureServices(services =>
    {
        var sp = services.BuildServiceProvider();

        using var scope = sp.CreateScope();

        var scopedServices = scope.ServiceProvider;
        var db = scopedServices.GetRequiredService<WeatherContext>();
        var logger = scopedServices.GetRequiredService<ILogger<WeatherApiWebApplicationFactory<TStartup>>>();

        db.Database.EnsureCreated();

        InitializeDbForTests(db);
    });
}

private static void InitializeDbForTests(WeatherContext db)
{
    db.WeatherForecasts.RemoveRange(db.WeatherForecasts);

    db.SaveChanges();

    db.WeatherForecasts.Add(new WeatherForecast(Guid.NewGuid(), "Australia/Melbourne",
        new DateTimeOffset(2020, 01, 02, 0, 0, 0, TimeSpan.Zero), 23.35m));

    db.SaveChanges();
}
```

```csharp
[Fact]
public async Task GetReturnsCorrectWeather()
{
    const string city = "Australia/Melbourne";
    const string forecastDate = "2020-01-02T00:00:00+00:00";
    var url = QueryHelpers.AddQueryString("/api/weather-forecasts", new Dictionary<string, string>
    {
        {"city", city},
        {"forecastDate", forecastDate}
    });

    using var client = _factory.CreateClient();
    using var response = await client.GetAsync(new Uri(url, UriKind.Relative));
    var responseContent = await response.Content.ReadAsStringAsync();
    var responseObj = JsonSerializer.Deserialize<object>(responseContent) as JsonElement?;

    Assert.Equal(city, responseObj?.GetProperty("city").ToString());
    Assert.Equal(forecastDate, responseObj?.GetProperty("forecastDate").ToString());
    Assert.Equal(23.35m,
        decimal.Parse(responseObj?.GetProperty("forecast").ToString()!, CultureInfo.InvariantCulture));
}
```

## Try it yourself

All of the code in this post is part of a sample application that I have hosted as a public repository on GitHub which you can access [here](https://github.com/gabrielsadaka/dotnet-production-ready). The readme contains steps on how to deploy it to your own GCP project if you would like to try it out. Look forward to hearing your thoughts/feedback in the comments below.
