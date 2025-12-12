# Workforce — Serviço HTTP (Blazor Client)

Este documento define as regras para a **camada de serviço HTTP no client** (Blazor WebAssembly) da entidade `<Entity>`:

* Interface `I<Entity>Service`.
* Implementação `<Entity>Service`.
* Registro no `Workforce.Client/Program.cs`.

Respeite:

* Persona, variáveis e regras gerais do documento principal  
      `Workforce.Specs/Prompts/Crud/CrudWithFrontend.md`
* Formato de saída em **Saída Esperada (formato)**.

---

## 1) Interfaces de Serviço

**Arquivos**: `<ServiceFiles>`

* `Workforce.Services.<Folder>.I<Entity>Service.cs`
* `Workforce.Services.<Folder>.<Entity>Service.cs`

### 1.1) Interface `I<Entity>Service`

A interface deve herdar de uma interface genérica, se existir:

```csharp
public interface I<Entity>Service : ICrudService<<Domain>>
{
    // Métodos adicionais específicos para Availability
}
```

Métodos padrão (além dos herdados de `ICrudService`, se aplicável):

* `Task<<Domain>?> GetByIdAsync(int id, CancellationToken ct = default);`
* `Task<IList<<Domain>>> GetAllAsync(CancellationToken ct = default);`
* `Task<<Domain>> InsertAsync(<Domain> entity, CancellationToken ct = default);`
* `Task<<Domain>?> UpdateAsync(<Domain> entity, CancellationToken ct = default);`
* `Task<bool> DeleteByIdAsync(int id, CancellationToken ct = default);`

Se `<Domain>` tiver `EnvironmentId`:

* `Task<IList<<Domain>>> GetAllByEnvironmentIdAsync(int environmentId, CancellationToken ct = default);`

---

---

## 2) Implementação `<Entity>Service`

**Classe**: `<Entity>Service : CrudService<<Domain>>, I<Entity>Service`

O serviço deve utilizar um `HttpClient` injetado.

* Serialização/deserialização JSON adequada (`System.Text.Json` ou outra biblioteca padrão do projeto).
* Deve chamar os endpoints definidos no controller (`/api/core/human-resource-management/availabilities/...` ou equivalente).

### 2.1) Métodos típicos

Exemplos de padrões (adaptar à infraestrutura já existente do projeto):

```csharp
public class AvailabilityService : CrudService<Availability>, IAvailabilityService
{
    private readonly HttpClient httpClient;

    public AvailabilityService(HttpClient httpClient) : base(httpClient)
    {
        this.httpClient = httpClient;
    }

    public async Task<Availability?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        var response = await httpClient.GetAsync($"api/core/human-resource-management/availabilities/{id}", ct);
        if (response.StatusCode == HttpStatusCode.NotFound)
        {
            return null;
        }

        response.EnsureSuccessStatusCode();
        var json = await response.Content.ReadAsStringAsync(ct);
        return JsonSerializer.Deserialize<Availability>(json, jsonOptions);
    }

    // Demais métodos (GetAllAsync, GetAllByEnvironmentIdAsync, InsertAsync, UpdateAsync, DeleteByIdAsync)
    // seguindo o mesmo padrão de tratamento de HTTP.
}
```

Adapte o código acima ao padrão de infraestrutura que já tiver (por exemplo, se `CrudService<T>` já fornece implementações padrão).

### 2.2) Tratamento de erros HTTP

* **404 NotFound** → retornar `null` quando fizer sentido (por exemplo, em `GetByIdAsync`).
* Outros status de erro (4xx, 5xx) → lançar `HttpRequestException` utilizando `response.EnsureSuccessStatusCode()` ou tratamento customizado.
* Propagar `CancellationToken` para todas as operações assíncronas.

Na UI (páginas Blazor), serão tratados casos específicos, por exemplo:

* `HttpRequestException` com `StatusCode == HttpStatusCode.InternalServerError` (500).
* Mensagens amigáveis ao utilizador via `NotificationService`.

---

## 3) Registro no Client (Workforce.Client/Program.cs)

Registe o serviço no container de DI:

```csharp
builder.Services.AddScoped<IAvailabilityService, AvailabilityService>();
```

ou, utilizando os placeholders:

```csharp
builder.Services.AddScoped<I<Entity>Service, <Entity>Service>();
```

As instâncias de `HttpClient` devem estar configuradas para apontar para o endpoint da API (por exemplo, `BaseAddress` do `HttpClient` configurada na aplicação Blazor).

---

## 4) Notas específicas de Serviço HTTP

* O service é a única camada responsável por dialogar com a API; as páginas Blazor não devem montar URLs manualmente (exceto em casos pontuais).
* Em operações de gravação (POST/PUT/DELETE), o service deve:
  * Encaminhar o `CancellationToken`.
  * Retornar a entidade inserida/atualizada ou booleano de sucesso.
* Exceções `HttpRequestException` propagadas pelo service devem ser tratadas nas páginas para:
  * Exibir notificações amigáveis.
  * Evitar mensagens falsas de erro quando a operação foi bem-sucedida no servidor, mas houve falha no response (caso de HTTP 500 após o save).