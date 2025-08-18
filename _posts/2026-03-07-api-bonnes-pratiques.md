---
title: "Bonnes pratiques pour développer une API .NET"
tags: dotnet api rest architecture bonnes-pratiques
---

Développer une API .NET robuste, maintenable et performante requiert bien plus qu'une simple mise en place d'un projet ASP.NET Core. Cet article rassemble les pratiques essentielles, de la conception à la mise en production.

<!--more-->

## 1. Concevoir une API RESTful cohérente

### Nommage des routes

Les routes doivent décrire des **ressources** (noms, jamais des verbes), en minuscules et avec des tirets :

```
GET    /api/orders           → liste des commandes
GET    /api/orders/{id}      → une commande
POST   /api/orders           → créer une commande
PUT    /api/orders/{id}      → remplacer une commande
PATCH  /api/orders/{id}      → modifier partiellement
DELETE /api/orders/{id}      → supprimer une commande
```

Évitez les routes du type `/api/getOrder` ou `/api/createOrder`.

### Codes HTTP corrects

| Situation | Code |
|-----------|------|
| Ressource retournée | `200 OK` |
| Ressource créée | `201 Created` + header `Location` |
| Traitement sans contenu | `204 No Content` |
| Données invalides | `400 Bad Request` |
| Non authentifié | `401 Unauthorized` |
| Accès interdit | `403 Forbidden` |
| Introuvable | `404 Not Found` |
| Erreur serveur | `500 Internal Server Error` |

---

## 2. Versionner son API

Le versioning permet de faire évoluer son API sans casser les clients existants.

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
});
```

Plusieurs stratégies existent : par URL (`/api/v1/orders`), par header (`api-version: 1.0`) ou par query string (`?api-version=1.0`). La stratégie par URL est la plus lisible et la plus répandue.

---

## 3. Séparer les modèles internes des contrats API

Ne jamais exposer directement les entités de domaine ou de base de données. Utiliser des **DTO (Data Transfer Objects)** dédiés :

```csharp
// Entité interne — ne pas exposer
public class Order
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public List<OrderLine> Lines { get; set; } = [];
    public decimal Total { get; set; }
    public OrderStatus Status { get; set; }
    public DateTime CreatedAt { get; set; }
}

// DTO de réponse
public record OrderResponse(
    Guid Id,
    decimal Total,
    string Status,
    DateTime CreatedAt
);

// DTO de création
public record CreateOrderRequest(
    Guid CustomerId,
    List<OrderLineRequest> Lines
);
```

Le mapping peut se faire manuellement ou via **Mapperly** (générateur de source, sans réflexion) :

```csharp
[Mapper]
public partial class OrderMapper
{
    public partial OrderResponse ToResponse(Order order);
}
```

---

## 4. Valider les entrées

### DataAnnotations (simple)

```csharp
public record CreateOrderRequest(
    [Required] Guid CustomerId,
    [MinLength(1)] List<OrderLineRequest> Lines
);
```

Activer la validation automatique avec `[ApiController]` sur le contrôleur.

### FluentValidation (recommandé pour les cas complexes)

```csharp
public class CreateOrderRequestValidator : AbstractValidator<CreateOrderRequest>
{
    public CreateOrderRequestValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Lines)
            .NotEmpty()
            .WithMessage("Une commande doit contenir au moins une ligne.");
        RuleForEach(x => x.Lines).SetValidator(new OrderLineRequestValidator());
    }
}
```

```csharp
builder.Services.AddValidatorsFromAssemblyContaining<CreateOrderRequestValidator>();
builder.Services.AddFluentValidationAutoValidation();
```

---

## 5. Gérer les erreurs de manière uniforme

### ProblemDetails (RFC 9457)

ASP.NET Core expose nativement le format `ProblemDetails`, qui est la norme pour les réponses d'erreur :

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.5",
  "title": "Not Found",
  "status": 404,
  "detail": "La commande 'abc-123' est introuvable.",
  "instance": "/api/orders/abc-123"
}
```

Activez le retour automatique de `ProblemDetails` :

```csharp
builder.Services.AddProblemDetails();
```

### Middleware de gestion des exceptions

Centralisez la gestion des exceptions non attrapées pour éviter de laisser fuir des stacktraces en production :

```csharp
app.UseExceptionHandler(exceptionHandlerApp =>
{
    exceptionHandlerApp.Run(async context =>
    {
        var exceptionHandlerFeature = context.Features.Get<IExceptionHandlerFeature>();
        var exception = exceptionHandlerFeature?.Error;

        context.Response.StatusCode = exception switch
        {
            NotFoundException  => StatusCodes.Status404NotFound,
            ValidationException => StatusCodes.Status400BadRequest,
            UnauthorizedException => StatusCodes.Status403Forbidden,
            _ => StatusCodes.Status500InternalServerError
        };

        await Results.Problem(
            detail: exception?.Message,
            statusCode: context.Response.StatusCode
        ).ExecuteAsync(context);
    });
});
```

---

## 6. Authentification et autorisation

### JWT

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!))
        };
    });
```

### Politiques d'autorisation

Préférez les **politiques** aux rôles bruts :

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy => policy.RequireRole("admin"));
    options.AddPolicy("OwnerOrAdmin", policy =>
        policy.RequireAssertion(ctx =>
            ctx.User.IsInRole("admin") || ctx.User.HasClaim("owner", "true")));
});
```

```csharp
[Authorize(Policy = "AdminOnly")]
[HttpDelete("{id}")]
public async Task<IActionResult> Delete(Guid id) { ... }
```

---

## 7. Limiter le débit (Rate Limiting)

Depuis .NET 7, le rate limiting est intégré :

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", limiterOptions =>
    {
        limiterOptions.PermitLimit = 100;
        limiterOptions.Window = TimeSpan.FromMinutes(1);
        limiterOptions.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        limiterOptions.QueueLimit = 10;
    });
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

app.UseRateLimiter();
```

```csharp
[EnableRateLimiting("fixed")]
[HttpGet]
public async Task<IActionResult> GetOrders() { ... }
```

---

## 8. Logging et observabilité

### Logging structuré

Utilisez `ILogger<T>` avec des propriétés nommées plutôt que de l'interpolation de chaîne :

```csharp
// À éviter
_logger.LogInformation($"Commande {orderId} créée par {userId}");

// Correct — permet l'indexation dans les outils de log (Seq, Loki, etc.)
_logger.LogInformation("Commande {OrderId} créée par {UserId}", orderId, userId);
```

### OpenTelemetry

Intégrez OpenTelemetry pour les traces distribuées, métriques et logs :

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter())
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddOtlpExporter());
```

---

## 9. Asynchronisme systématique

Chaque opération I/O (base de données, appels HTTP, fichiers) **doit** être asynchrone :

```csharp
// À éviter
public IActionResult GetOrder(Guid id)
{
    var order = _repository.GetById(id); // bloque un thread
    return Ok(order);
}

// Correct
public async Task<IActionResult> GetOrder(Guid id)
{
    var order = await _repository.GetByIdAsync(id);
    return order is null ? NotFound() : Ok(order);
}
```

Passez également les `CancellationToken` du contrôleur jusqu'aux appels les plus profonds :

```csharp
public async Task<IActionResult> GetOrder(Guid id, CancellationToken ct)
{
    var order = await _repository.GetByIdAsync(id, ct);
    return order is null ? NotFound() : Ok(order);
}
```

---

## 10. Caching

### Cache en mémoire

```csharp
builder.Services.AddMemoryCache();

// Dans le service
public async Task<OrderResponse?> GetOrderAsync(Guid id, CancellationToken ct)
{
    return await _cache.GetOrCreateAsync($"order:{id}", async entry =>
    {
        entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
        var order = await _repository.GetByIdAsync(id, ct);
        return order is null ? null : _mapper.ToResponse(order);
    });
}
```

### Cache de réponse HTTP

```csharp
builder.Services.AddOutputCache();
app.UseOutputCache();

[OutputCache(Duration = 60)]
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(Guid id, CancellationToken ct) { ... }
```

---

## 11. Documentation avec OpenAPI

Depuis .NET 9, le support natif d'OpenAPI est amélioré :

```csharp
builder.Services.AddOpenApi();

app.MapOpenApi(); // expose /openapi/v1.json
```

Enrichissez la documentation avec des attributs :

```csharp
/// <summary>Récupère une commande par son identifiant.</summary>
/// <param name="id">L'identifiant de la commande.</param>
/// <response code="200">La commande demandée.</response>
/// <response code="404">Commande introuvable.</response>
[HttpGet("{id}")]
[ProducesResponseType<OrderResponse>(StatusCodes.Status200OK)]
[ProducesResponseType<ProblemDetails>(StatusCodes.Status404NotFound)]
public async Task<IActionResult> GetOrder(Guid id, CancellationToken ct) { ... }
```

---

## 12. Tests

### Tests unitaires

Testez les services et la logique métier en isolation avec **xUnit** et **NSubstitute** :

```csharp
public class OrderServiceTests
{
    private readonly IOrderRepository _repository = Substitute.For<IOrderRepository>();
    private readonly OrderService _sut;

    public OrderServiceTests() => _sut = new OrderService(_repository);

    [Fact]
    public async Task GetOrder_ReturnsNull_WhenNotFound()
    {
        _repository.GetByIdAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>())
                   .Returns((Order?)null);

        var result = await _sut.GetOrderAsync(Guid.NewGuid(), default);

        result.Should().BeNull();
    }
}
```

### Tests d'intégration

Utilisez `WebApplicationFactory<TProgram>` pour tester les endpoints de bout en bout :

```csharp
public class OrdersControllerTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public OrdersControllerTests(WebApplicationFactory<Program> factory)
        => _client = factory.CreateClient();

    [Fact]
    public async Task GetOrder_Returns404_WhenNotFound()
    {
        var response = await _client.GetAsync($"/api/orders/{Guid.NewGuid()}");
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
}
```

---

## Récapitulatif

| Pratique | Outil / Approche |
|----------|-----------------|
| Routes RESTful | Conventions de nommage + verbes HTTP |
| Versioning | `Asp.Versioning.Http` |
| Modèles d'échange | DTO + Mapperly |
| Validation | FluentValidation |
| Erreurs uniformes | `ProblemDetails` + middleware |
| Authentification | JWT Bearer |
| Rate limiting | `RateLimiter` intégré (.NET 7+) |
| Observabilité | OpenTelemetry + logging structuré |
| Performance | `async`/`await` + `CancellationToken` |
| Caching | `IMemoryCache` ou `OutputCache` |
| Documentation | OpenAPI natif (.NET 9+) |
| Tests | xUnit + NSubstitute + `WebApplicationFactory` |

Une API bien conçue ne s'arrête pas à faire fonctionner des endpoints : elle doit être **prévisible**, **sécurisée**, **observable** et **testable** dès le départ.
