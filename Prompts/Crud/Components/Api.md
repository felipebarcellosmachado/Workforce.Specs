# Workforce — Availability — API (Controller & Server Registration)

Este documento define as regras para a **camada de API** da entidade `<Entity>` (Availability):

* Controller Web API para `<Entity>`.
* Rotas, códigos de status e convenções.
* Registro do repositório e `DbContext` no `Workforce.Server`.

Respeite sempre:

* Persona, variáveis e regras gerais do documento principal  
  `docs/Workforce.Availability.CRUD.Main.md`.
* Formato de saída em **Saída Esperada (formato)** do documento principal.

---

## 1) Controller (Web API)

**Arquivo**: `<ControllerFile>`

### 1.1) Atributos / Decorators

Adicione os atributos:

```csharp
[ApiController]
[Route("api/[controller]")]
```

Ou defina uma rota explícita coerente com a área, por exemplo:

```csharp
[Route("api/core/human-resource-management/[controller]")]
```

Escolha uma rota consistente com o resto da API, por exemplo:

```
api/core/human-resource-management/availabilities
```

### 1.2) Dependência injetada

O controller deve receber via DI:

* Uma instância de `<Entity>Repository`.

Exemplo (esqueleto):

```csharp
[ApiController]
[Route("api/core/human-resource-management/availabilities")]
public class AvailabilityController : ControllerBase
{
    private readonly AvailabilityRepository repository;

    public AvailabilityController(AvailabilityRepository repository)
    {
        this.repository = repository;
    }

    // Métodos...
}
```

### 1.3) Endpoints obrigatórios

#### a) GET por Id

```csharp
[HttpGet("{id:int}", Name = "Get<Entity>ById")]
public async Task<ActionResult<<Domain>>> GetByIdAsync(int id, CancellationToken ct = default)
```

* Retorna **200 OK** com a entidade se encontrada.
* Retorna **404 Not Found** se não encontrada.
* A propriedade `Name = "Get<Entity>ById"` é importante para roteamento (mesmo que não use `CreatedAtAction`).

#### b) Endpoints com EnvironmentId (se `<Domain>` possuir esta propriedade)

**Obter por ambiente e Id:**

```csharp
[HttpGet("environment/{environmentId:int}/{id:int}")]
public async Task<ActionResult<<Domain>>> GetByEnvironmentIdAndIdAsync(int environmentId, int id, CancellationToken ct = default)
```

* Opcional: validar se o id pertence ao `environmentId` e retornar 404 se não pertencer.

**Obter todos por ambiente:**

```csharp
[HttpGet("all/environment/{environmentId:int}")]
public async Task<ActionResult<IList<<Domain>>>> GetAllByEnvironmentIdAsync(int environmentId, CancellationToken ct = default)
```

#### c) Endpoint de lista sem EnvironmentId (se não existir esta propriedade)

Caso a entidade não tenha `EnvironmentId`:

```csharp
[HttpGet("all")]
public async Task<ActionResult<IList<<Domain>>>> GetAllAsync(CancellationToken ct = default)
```

#### d) POST — Insert

```csharp
[HttpPost]
public async Task<ActionResult<<Domain>>> InsertAsync([FromBody] <Domain> entity, CancellationToken ct = default)
```

* Valida o payload (`ModelState`).
* Chama `InsertAsync` do repositório.
* Deve retornar **201 Created** com header `Location`.

**CRÍTICO** – usar `Created` em vez de `CreatedAtAction`:

Para evitar o erro de rota "No route matches the supplied values", use uma URI explícita:

```csharp
var insertedEntity = await repository.InsertAsync(entity, ct);

// Forma 1 (baseada na rota explícita):
return Created($"/api/core/human-resource-management/availabilities/{insertedEntity.Id}", insertedEntity);

// ou, forma 2 (baseada no Request.Path):
return Created($"{Request.Path}/{insertedEntity.Id}", insertedEntity);
```

#### e) PUT — Update

```csharp
[HttpPut("{id:int}")]
public async Task<ActionResult<<Domain>>> UpdateAsync(int id, [FromBody] <Domain> entity, CancellationToken ct = default)
```

* Garante que `id == entity.Id`; caso contrário, retorna **400 Bad Request**.
* Chama `UpdateAsync` do repositório.
* Se o repositório retornar `null`, significa que a entidade não foi encontrada → **404 Not Found**.
* Se atualizado com sucesso, retorna **200 OK** com a entidade atualizada.

#### f) DELETE — Delete

```csharp
[HttpDelete("{id:int}")]
public async Task<IActionResult> DeleteByIdAsync(int id, CancellationToken ct = default)
```

* Chama `DeleteByIdAsync` do repositório.
* Se retornar `false` → **404 Not Found**.
* Se retornar `true` → **204 No Content**.

### 1.4) Códigos de resposta esperados

* **200 OK** – em GET por Id/GET lista/PUT com sucesso.
* **201 Created** – em POST (insert bem-sucedido).
* **204 No Content** – em DELETE bem-sucedido.
* **400 Bad Request** – para payload inválido ou inconsistência entre rota e corpo (id divergente).
* **404 Not Found** – quando o recurso não é encontrado.

---

## 2) Registro no Server (Workforce.Server/Program.cs)

Certifique-se de registrar o repositório e o contexto de banco de dados:

```csharp
services.AddScoped<AvailabilityRepository>();
```

Se o contexto ainda não estiver registrado:

```csharp
services.AddDbContext<WorkforceDbContext>(options =>
    options.UseNpgsql(connectionString));
```

A `connectionString` deve apontar para o banco PostgreSQL adequado.

---

## 3) Erros Comuns de API e Boas Práticas

### 3.1) Erro: "No route matches the supplied values"

**Causa típica:**

Uso de:

```csharp
return CreatedAtAction(nameof(GetByIdAsync), new { id = insertedEntity.Id }, insertedEntity);
```

sem que a rota corresponda exatamente aos parâmetros esperados, ou com rota personalizada não compatível.

**Solução recomendada:**

Utilizar `Created` com URI explícita, por exemplo:

```csharp
return Created($"/api/core/human-resource-management/availabilities/{insertedEntity.Id}", insertedEntity);
```

ou:

```csharp
return Created($"{Request.Path}/{insertedEntity.Id}", insertedEntity);
```

### 3.2) Outras boas práticas

* Em controllers, respeitar o uso de `CancellationToken` propagando-o às chamadas do repositório.
* Validar `ModelState` e retornar 400 quando necessário.
* Em cenários de erro interno, retornar 500 apropriado, permitindo que o client trate `HttpRequestException`.