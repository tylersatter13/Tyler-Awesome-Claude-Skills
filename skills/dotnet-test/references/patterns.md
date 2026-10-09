# MSTest patterns

Adapt these to the repo's existing setup. If the repo already has a fixture or base class, use it.

## Packages

```xml
<PackageReference Include="MSTest" />                       <!-- MSTest 3.x meta-package -->
<PackageReference Include="AwesomeAssertions" />           <!-- readable .Should() assertions -->
<PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" />
<PackageReference Include="Testcontainers.MsSql" />
<PackageReference Include="Testcontainers.Azurite" />
<PackageReference Include="WireMock.Net" />
<PackageReference Include="Respawn" />                      <!-- optional: DB reset between tests -->
```

Or use the MSTest SDK project style: `<Project Sdk="MSTest.Sdk/3.x">`.

Assertions use AwesomeAssertions (`using AwesomeAssertions;`), the community fork of
FluentAssertions: `actual.Should().Be(expected)`. Prefer it over MSTest `Assert` in new tests, and
add a reason string when the failure would otherwise be unclear. If a repo already uses
FluentAssertions, the API is the same; keep what is there.

The API project needs `public partial class Program;` at the end of `Program.cs` (or
`InternalsVisibleTo` for the test assembly) so `WebApplicationFactory<Program>` can see it.

## Shared containers and app factory (once per test assembly)

```csharp
[TestClass]
public static class TestHost
{
    private static readonly MsSqlContainer Sql = new MsSqlBuilder().Build();
    public static WireMockServer Partner { get; private set; } = null!;
    public static ApiFactory Factory { get; private set; } = null!;

    [AssemblyInitialize]
    public static async Task Init(TestContext _)
    {
        await Sql.StartAsync();
        Partner = WireMockServer.Start();
        Factory = new ApiFactory(Sql.GetConnectionString(), Partner.Url!);

        using var scope = Factory.Services.CreateScope();
        await scope.ServiceProvider.GetRequiredService<AppDbContext>().Database.MigrateAsync();
    }

    [AssemblyCleanup]
    public static async Task Cleanup()
    {
        await Factory.DisposeAsync();
        Partner.Stop();
        await Sql.DisposeAsync();
    }
}

public sealed class ApiFactory(string connectionString, string partnerUrl)
    : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");
        builder.UseSetting("ConnectionStrings:Default", connectionString);
        builder.UseSetting("Partner:BaseUrl", partnerUrl);
        builder.ConfigureTestServices(services =>
        {
            // Replace auth with a test scheme, e.g. one that reads claims from a header.
        });
    }
}
```

## API test

```csharp
[TestClass]
public sealed class OrdersApiTests
{
    public TestContext TestContext { get; set; } = null!;

    [TestMethod]
    [Description("AC2")]
    public async Task Post_Orders_WithMissingSku_Returns400()
    {
        var client = TestHost.Factory.CreateClient();

        var response = await client.PostAsJsonAsync(
            "/api/v1/orders", new { quantity = 1 }, TestContext.CancellationTokenSource.Token);

        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
        var problem = await response.Content.ReadFromJsonAsync<ValidationProblemDetails>();
        problem!.Errors.Should().ContainKey("Sku");
    }
}
```

## Partner failure with WireMock.Net

```csharp
[TestInitialize]
public void ResetPartner() => TestHost.Partner.Reset();

[TestMethod]
public async Task Sync_WhenPartnerReturns503_RetriesThenReturns502()
{
    TestHost.Partner
        .Given(Request.Create().WithPath("/v2/refunds").UsingPost())
        .RespondWith(Response.Create().WithStatusCode(503));

    var response = await TestHost.Factory.CreateClient().PostAsync("/api/v1/refunds/sync", null);

    response.StatusCode.Should().Be(HttpStatusCode.BadGateway);
    TestHost.Partner.LogEntries.Should().HaveCountGreaterThan(1, "the resilience handler should retry 503s");
}
```

Other cases to cover: `.WithDelay(...)` past the client timeout, 429 with a `Retry-After`
header, and a 200 with a malformed body.

## Data-driven unit test

```csharp
[TestMethod]
[DataRow(0, false)]
[DataRow(1, true)]
[DataRow(1000, true)]
[DataRow(1001, false)]
public void IsValidQuantity_ReturnsExpected(int quantity, bool expected) =>
    OrderRules.IsValidQuantity(quantity).Should().Be(expected);
```
