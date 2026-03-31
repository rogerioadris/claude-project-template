# Padrões de Teste

## Objetivo

Guia para escrever testes unitários e de integração no backend (.NET) e frontend (Angular).

## Quando usar

- Criar testes para nova feature
- Adicionar cobertura a código existente
- Definir estratégia de teste para um módulo

---

## Backend — Testes Unitários (xUnit + Moq + FluentAssertions)

### Command Handler

```csharp
public sealed class CreateExampleCommandHandlerTests
{
    private readonly Mock<IExampleRepository> _repositoryMock = new();
    private readonly Mock<IAuditService> _auditServiceMock = new();
    private readonly Mock<IUnitOfWork> _unitOfWorkMock = new();
    private readonly CreateExampleCommandHandler _handler;

    public CreateExampleCommandHandlerTests()
    {
        _handler = new CreateExampleCommandHandler(
            _repositoryMock.Object,
            _auditServiceMock.Object,
            _unitOfWorkMock.Object
        );
    }

    [Fact]
    public async Task Handle_ValidCommand_ReturnsGuid()
    {
        // Arrange
        var command = new CreateExampleCommand("Nome", "Descrição");
        _repositoryMock
            .Setup(r => r.ExistsByNameAsync(command.Name, It.IsAny<CancellationToken>()))
            .ReturnsAsync(false);

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        result.IsError.Should().BeFalse();
        result.Value.Should().NotBeEmpty();
        _unitOfWorkMock.Verify(u => u.SaveChangesAsync(It.IsAny<CancellationToken>()), Times.Once);
        _auditServiceMock.Verify(a => a.LogAsync(
            "CREATE", "Examples", It.IsAny<string>(),
            It.IsAny<string?>(), It.IsAny<string?>(),
            It.IsAny<CancellationToken>()), Times.Once);
    }

    [Fact]
    public async Task Handle_DuplicateName_ReturnsConflictError()
    {
        // Arrange
        var command = new CreateExampleCommand("Existente", "Descrição");
        _repositoryMock
            .Setup(r => r.ExistsByNameAsync(command.Name, It.IsAny<CancellationToken>()))
            .ReturnsAsync(true);

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        result.IsError.Should().BeTrue();
        result.FirstError.Type.Should().Be(ErrorType.Conflict);
    }
}
```

### Query Handler

```csharp
[Fact]
public async Task Handle_ExistingId_ReturnsDto()
{
    var example = new Example("Nome", "Desc");
    _repositoryMock
        .Setup(r => r.GetByIdAsync(example.Id, It.IsAny<CancellationToken>()))
        .ReturnsAsync(example);

    var result = await _handler.Handle(new GetExampleByIdQuery(example.Id), CancellationToken.None);

    result.IsError.Should().BeFalse();
    result.Value.Name.Should().Be("Nome");
}

[Fact]
public async Task Handle_NonExistingId_ReturnsNotFound()
{
    _repositoryMock
        .Setup(r => r.GetByIdAsync(It.IsAny<Guid>(), It.IsAny<CancellationToken>()))
        .ReturnsAsync((Example?)null);

    var result = await _handler.Handle(new GetExampleByIdQuery(Guid.NewGuid()), CancellationToken.None);

    result.IsError.Should().BeTrue();
    result.FirstError.Type.Should().Be(ErrorType.NotFound);
}
```

### Validator

```csharp
public sealed class CreateExampleCommandValidatorTests
{
    private readonly CreateExampleCommandValidator _validator = new();

    [Fact]
    public void Validate_EmptyName_Fails()
    {
        var command = new CreateExampleCommand("", "Descrição");
        var result = _validator.Validate(command);
        result.IsValid.Should().BeFalse();
        result.Errors.Should().Contain(e => e.PropertyName == "Name");
    }

    [Fact]
    public void Validate_ValidCommand_Passes()
    {
        var command = new CreateExampleCommand("Nome", "Descrição");
        _validator.Validate(command).IsValid.Should().BeTrue();
    }
}
```

---

## Backend — Testes de Integração (WebApplicationFactory)

```csharp
public sealed class ExamplesControllerTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public ExamplesControllerTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Substituir DbContext por banco in-memory ou container de teste
            });
        }).CreateClient();
    }

    [Fact]
    public async Task GetAll_ReturnsOk()
    {
        var response = await _client.GetAsync("/api/examples");
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }

    [Fact]
    public async Task Create_WithoutAuth_Returns401()
    {
        var response = await _client.PostAsJsonAsync("/api/examples", new { Name = "Test" });
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }
}
```

### Fluxos obrigatórios para Controller

- [ ] Sucesso (200/201)
- [ ] Não encontrado (404)
- [ ] Não autorizado (401)
- [ ] Sem permissão (403)
- [ ] Conflito/duplicata (409)
- [ ] Validação inválida (422)

---

## Backend — Testes de Integração com Testcontainers

Alternativa ao banco in-memory: containers Docker reais para testes mais fiéis ao ambiente de produção.

```csharp
// NuGet: Testcontainers.PostgreSql
public sealed class IntegrationTestFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.AddDbContext<AppDbContext>(options =>
                options.UseNpgsql(_postgres.GetConnectionString()));
        });
    }

    public async Task InitializeAsync() => await _postgres.StartAsync();
    public async Task DisposeAsync() => await _postgres.DisposeAsync();
}

// Uso
public sealed class UsersControllerTests(IntegrationTestFactory factory)
    : IClassFixture<IntegrationTestFactory>
{
    private readonly HttpClient _client = factory.CreateClient();

    [Fact]
    public async Task CreateUser_ReturnsCreated()
    {
        var response = await _client.PostAsJsonAsync("/api/v1/users", new { Name = "João", Email = "joao@email.com" });
        response.StatusCode.Should().Be(HttpStatusCode.Created);
    }

    [Fact]
    public async Task GetAll_ReturnsOk()
    {
        var response = await _client.GetAsync("/api/v1/users");
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }
}
```

> **Importante:** Testcontainers requer Docker rodando localmente. Cada classe de teste cria um container PostgreSQL isolado, garantindo que os testes não interferem entre si.

---

## Frontend — Testes Unitários (Jasmine + TestBed)

### Service

```typescript
describe('ExampleService', () => {
  let service: ExampleService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(),
        provideHttpClientTesting(),
      ],
    });
    service = TestBed.inject(ExampleService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpMock.verify());

  it('should list examples', () => {
    const mockData: ExamplePaginated = { data: [], total: 0, page: 1, lastPage: 1 };

    service.listar().subscribe(result => {
      expect(result.total).toBe(0);
    });

    const req = httpMock.expectOne(r => r.url.includes('/examples'));
    expect(req.request.method).toBe('GET');
    req.flush(mockData);
  });
});
```

### Standalone Component

```typescript
describe('ExampleListComponent', () => {
  let fixture: ComponentFixture<ExampleListComponent>;
  let component: ExampleListComponent;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [ExampleListComponent],
      providers: [
        { provide: ExampleService, useValue: jasmine.createSpyObj('ExampleService', ['listar']) },
      ],
    }).compileComponents();

    fixture = TestBed.createComponent(ExampleListComponent);
    component = fixture.componentInstance;
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should have OnPush change detection', () => {
    const metadata = (ExampleListComponent as any).ɵcmp;
    expect(metadata.changeDetection).toBe(ChangeDetectionStrategy.OnPush);
  });
});
```

---

## Quando Usar Mock vs Real

| Cenário | Mock | Real |
|---------|------|------|
| Handler unitário | Mock repositório, audit, UoW | — |
| Validator | Nenhum mock necessário | — |
| Controller integração | — | WebApplicationFactory + DB in-memory |
| Service Angular | HttpTestingController | — |
| Component Angular | Mock do service | — |

---

## Comandos

```bash
# Backend — rodar todos
dotnet test

# Backend — filtrar por feature
dotnet test --filter "FullyQualifiedName~{Feature}"

# Frontend — rodar todos
npm test

# Frontend — com cobertura
npm test -- --code-coverage
```

## Cobertura Mínima

- **Services Angular:** 80%
- **Handlers backend:** 100% dos fluxos (sucesso + cada tipo de erro)
- **Validators:** 100% das regras
