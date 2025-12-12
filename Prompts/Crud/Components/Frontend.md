# Workforce — Frontend (Blazor + Radzen + Layout)

Este documento define as regras para a **camada de Frontend** da entidade `<Entity>`:

* Página de **Lista** (`<ListPage>`).
* Página de **Edição** (`<EditPage>`).
* **MainLayout** (menu).
* Diretrizes de **UX/UI** com Radzen Material 3.
* Padronização de validações e tratamento de erros na UI.

Respeite:

* Persona, variáveis e regras gerais do documento principal  
      `Workforce.Specs/Prompts/Crud/CrudWithFrontend.md`
* Formato de saída descrito em **Saída Esperada (formato)**.

---

## 1) Página de Lista (`<ListPage>`)

**Arquivos**:  

* `<ListPage>.razor`
* `<ListPage>.razor.cs` (herda de `ComponentBase`)

Use como referência visual a página `OrganizationsPage`.

### 1.1) Estrutura de layout obrigatória

No `.razor`, utilize um container raiz:

```razor
<RadzenStack Orientation="Orientation.Vertical" Gap="1rem" Style="width: 100%;">
    <!-- Todo conteúdo aqui -->
</RadzenStack>
```

Dentro deste container:

* Barra de título (ex.: `<h3>@L["<Entities>"]</h3>` ou `RadzenText` com `TextStyle.H4`).
* Barra de ações (botões ícones para Novo, Refresh, Export, etc.).
* `RadzenDataGrid<<Domain>>` para listar as entidades.

### 1.2) RadzenDataGrid

Configure o `RadzenDataGrid<<Domain>>` com:

* `Data="@items"`
* `AllowFiltering="true"`
* `AllowSorting="true"`
* `AllowPaging="true"`
* `RowDoubleClick="OnRowDoubleClick"` para navegar à página de edição.
* `Style="width: 100%;"` para ocupar toda a largura disponível.

### 1.3) Injeções no code-behind

No arquivo `.razor.cs`:

```csharp
[Inject] public IAppState AppState { get; set; } = default!;
[Inject] public IStringLocalizer<SharedResources> L { get; set; } = default!;
[Inject] public I<Entity>Service Service { get; set; } = default!;
[Inject] public NavigationManager Nav { get; set; } = default!;
```

### 1.4) Carregamento de dados

Em `OnInitializedAsync()` ou `OnParametersSetAsync()`:

**Se a entidade tiver `EnvironmentId`:**

```csharp
items = await Service.GetAllByEnvironmentIdAsync(AppState.Session.Environment.Id, ct);
```

**Caso contrário:**

```csharp
items = await Service.GetAllAsync(ct);
```

### 1.5) Toolbar de ações

Logo acima do `DataGrid`, crie uma barra de ações:

* **Botão Novo**: navega para `<EditPage>` com `Id = 0`.
* **Botão Refresh**: recarrega os dados.
* **Botão Export** (opcional): exporta a lista (para Excel/CSV, se aplicável).

Use apenas ícones nos botões; o texto pode ser exibido via tooltip.

---

## 2) Página de Edição (`<EditPage>`)

**Arquivos:**

* `<EditPage>.razor`
* `<EditPage>.razor.cs` (herda de `ComponentBase`)

Use como referência visual a página `OrganizationPage`.

### 2.1) Parâmetros e injeções

No `.razor.cs`:

```csharp
[Parameter] public int Id { get; set; }

[Inject] public IAppState AppState { get; set; } = default!;
[Inject] public IStringLocalizer<SharedResources> L { get; set; } = default!;
[Inject] public I<Entity>Service Service { get; set; } = default!;
[Inject] public NavigationManager Nav { get; set; } = default!;
// Injetar serviços para foreign keys quando necessário
```

Importe também:

```csharp
using System.Net; // Para HttpStatusCode
```

### 2.2) Estrutura de layout obrigatória

No `.razor`, use:

```razor
<RadzenStack Orientation="Orientation.Vertical" Gap="1rem" Style="width: 100%;">
    <!-- Context Bar -->
    <RadzenCard>
        <RadzenStack Orientation="Orientation.Horizontal"
                     JustifyContent="JustifyContent.SpaceBetween"
                     AlignItems="AlignItems.Center">
            <RadzenText TextStyle="TextStyle.H4" Text="@title" />
            <RadzenStack Orientation="Orientation.Horizontal" Gap="0.5rem">
                <!-- Botões de ação (Salvar, Voltar, etc.) -->
            </RadzenStack>
        </RadzenStack>
    </RadzenCard>

    <!-- Content Cards -->
    <RadzenCard>
        <!-- Campos do formulário -->
    </RadzenCard>
</RadzenStack>
```

**NUNCA** use, como raiz, algo do tipo:

```razor
<RadzenRow class="rz-background-color-base-600">
    <RadzenColumn Size="10">...</RadzenColumn>
    <RadzenColumn Size="2">...</RadzenColumn>
</RadzenRow>
<RadzenRow Style="margin-top: 10px;">
    <RadzenColumn Size="12">...</RadzenColumn>
</RadzenRow>
```

Prefira sempre `RadzenStack` com `RadzenCard`.

### 2.3) Ciclo de vida e inicialização

No `OnInitializedAsync()` ou `OnParametersSetAsync()`:

**Se `Id > 0`:**

Carregar entidade existente:

```csharp
entity = await Service.GetByIdAsync(Id, ct);
if (entity == null)
{
    // Redirecionar para lista ou exibir erro apropriado
}
```

**Se `Id == 0`:**

Criar nova instância:

```csharp
entity = new <Domain>();
```

**Se houver `EnvironmentId`:**

```csharp
entity.EnvironmentId = AppState.Session.Environment.Id;
```

Carregar listas para foreign keys (dropdowns), usando `GetAllAsync` ou `GetAllByEnvironmentIdAsync` conforme o caso.

### 2.4) Formulário e campos

* Use `RadzenTemplateForm` para o formulário principal.
* Se a entidade tiver muitas propriedades, divida em múltiplos `RadzenCard` ou use `RadzenTabs` com `RadzenTabsItem`.

Mapeie os campos conforme o tipo:

* **Texto**: `RadzenTextBox`.
* **Numéricos**: `RadzenNumeric<T>`.
* **Booleanos**: `RadzenCheckBox`.
* **Datas**:
  * `RadzenDatePicker` para `DateOnly` ou `DateTime`.
  * Para `DateTime`, defina formato de data e hora (DD/MM/YYYY HH:mm:ss).
* **Enums**:
  * `RadzenDropDown` com `Enum.GetValues(typeof(SeuEnum))`.
* **Foreign Keys** (? `EnvironmentId`):
  * `RadzenDropDown` com dados carregados via service.
  * Se a entidade estrangeira tiver `EnvironmentId`, use o `GetAllByEnvironmentIdAsync` com `<EnvIdExpr>`.

### 2.5) Validadores e erro RZ9996

**ERRO COMUM (RZ9996)**

Não coloque validadores dentro de `RadzenFormField` usando `<ChildContent>`.

**Forma incorreta (NÃO usar):**

```razor
<RadzenFormField Text="@L["Name"]" Variant="Variant.Outlined">
    <RadzenTextBox @bind-Value="@entity.Name" Name="Name" />
    <ChildContent>
        <RadzenRequiredValidator Component="Name" Text="@L["NameRequired"]" />
    </ChildContent>
</RadzenFormField>
```

**Forma correta (sempre usar):**

```razor
<RadzenFormField Text="@L["Name"]" Variant="Variant.Outlined">
    <RadzenTextBox @bind-Value="@entity.Name" Name="Name" Style="width: 100%;" />
</RadzenFormField>
<RadzenRequiredValidator Component="Name" Text="@L["NameRequired"]" />
```

Ou seja, os validadores ficam **fora** do `RadzenFormField`.

### 2.6) Tratamento de exceções no Save (CRÍTICO)

Na ação de salvar (por exemplo, `SaveAsync` ou similar), trate casos de HTTP 500 após o save, para evitar mensagens de erro falsas.

**Estrutura sugerida:**

```csharp
private async Task SaveAvailabilityAsync()
{
    try
    {
        // Validações locais...

        if (Id > 0)
        {
            var updatedEntity = await Service.UpdateAsync(entity);
            NotificationService.Notify(new NotificationMessage
            {
                Severity = NotificationSeverity.Success,
                Summary = L["Success"],
                Detail = L["UpdateSuccess"]
            });
            NavigationManager.NavigateTo("/core/human-resource-management/availabilities");
        }
        else
        {
            var newEntity = await Service.InsertAsync(entity);
            NotificationService.Notify(new NotificationMessage
            {
                Severity = NotificationSeverity.Success,
                Summary = L["Success"],
                Detail = L["CreateSuccess"]
            });
            NavigationManager.NavigateTo("/core/human-resource-management/availabilities");
        }
    }
    catch (HttpRequestException httpEx) when (httpEx.StatusCode == HttpStatusCode.InternalServerError)
    {
        // Dados podem ter sido salvos, mas houve erro no response
        Console.WriteLine($"HTTP 500 after save: {httpEx.Message}");
        NotificationService.Notify(new NotificationMessage
        {
            Severity = NotificationSeverity.Warning,
            Summary = L["Warning"],
            Detail = L["DataSavedButResponseError"]
        });
        NavigationManager.NavigateTo("/core/human-resource-management/availabilities");
    }
    catch (HttpRequestException httpEx)
    {
        NotificationService.Notify(new NotificationMessage
        {
            Severity = NotificationSeverity.Error,
            Summary = L["Error"],
            Detail = $"{L["SaveError"]}: HTTP {(int?)httpEx.StatusCode ?? 0} - {httpEx.Message}"
        });
    }
    catch (Exception ex)
    {
        NotificationService.Notify(new NotificationMessage
        {
            Severity = NotificationSeverity.Error,
            Summary = L["Error"],
            Detail = $"{L["SaveError"]}: {ex.Message}"
        });
    }
}
```

Ajuste rota/nome da página de lista conforme o path real da aplicação.

### 2.7) Botões de ação (ícones)

Na "Context Bar" (card superior da página de edição), inclua botões somente com ícones:

**Salvar:**

* `ButtonStyle.Success`
* `Icon="save"`
* Chama `SaveAvailabilityAsync()`.

**Voltar:**

* `ButtonStyle.Secondary`
* `Icon="arrow_back"`
* Navega para a lista (`<ListPage>`).

---

## 3) MainLayout (Menu)

**Arquivo**: `Workforce.Client/Shared/MainLayout.razor` (ou equivalente)

Adicione a entrada de menu para a página de lista de `<Entity>`:

```razor
<RadzenPanelMenuItem Text="@L["<Entities>"]"
                     Icon="list"
                     Path="/core/human-resource-management/availabilities" />
```

Ajuste o `Path` de acordo com a rota real do `<ListPage>`.

Evite criar link direto para `<EditPage>`; o acesso deve ser feito via lista (ou navegação programática).

---

## 4) UX/UI (Diretrizes Radzen Material 3)

### 4.1) Estrutura de página

* **SEMPRE** use `RadzenStack` como container raiz com `Style="width: 100%;"`.
* Conteúdo organizado em `RadzenCard` dentro do `RadzenStack`.
* `RadzenDataGrid` com `Style="width: 100%;"`.
* Evite tamanhos fixos (`width: 800px`, `height: 100vh`, etc.).

### 4.2) Barra superior (Contexto / Título)

Use `RadzenCard` contendo `RadzenStack Orientation="Horizontal"`:

* **À esquerda**: título (`RadzenText` com `TextStyle.H4` ou `<h3>`).
* **À direita**: botões de ação (ícones).

### 4.3) Barra de ações

Logo abaixo do título, use uma barra com botões de ação:

* **Salvar**: `ButtonStyle.Success`, `Icon="save"`.
* **Voltar**: `ButtonStyle.Secondary`, `Icon="arrow_back"`.
* **Novo** (na lista): `ButtonStyle.Primary`, `Icon="add"`.
* **Refresh**: `ButtonStyle.Light`, `Icon="refresh"`.

Os textos podem ser exibidos via tooltip, mantendo a interface mais limpa.

### 4.4) Responsividade

* Use `Gap` para espaçamento entre componentes (`Gap="1rem"` ou similar).
* Use `RadzenRow`/`RadzenColumn` apenas dentro de cards para layouts mais complexos de campos.
* Inputs (`RadzenTextBox`, `RadzenNumeric`, etc.) com `Style="width: 100%;"`.

### 4.5) Internacionalização

Todo texto visível deve usar `IStringLocalizer<SharedResources>`, por exemplo:

* `L["Availability"]`
* `L["CreateSuccess"]`
* `L["UpdateSuccess"]`
* `L["SaveError"]`, etc.

---

## 5) Erros Comuns no Frontend

### 5.1) RZ9996 — "Unrecognized child content inside component 'RadzenFormField'"

**Causa:**

Validadores colocados como `<ChildContent>` dentro de `RadzenFormField`.

**Solução:**

Sempre colocar validadores (`RadzenRequiredValidator`, etc.) **fora** de `RadzenFormField`.  
Ver secção **2.5) Validadores e erro RZ9996**.

### 5.2) Mensagem de erro mesmo com dados salvos

**Causa:**

Em alguns cenários, o servidor pode salvar os dados e, ainda assim, retornar HTTP 500 (por exemplo, erro ao serializar resposta ou outro problema pós-persistência).

A UI, se não tratar `HttpRequestException` com `StatusCode == HttpStatusCode.InternalServerError`, exibirá erro ao utilizador, embora os dados já estejam no banco.

**Solução:**

Tratar `HttpRequestException` com filtro para `StatusCode == HttpStatusCode.InternalServerError` na rotina de save (ver secção **2.6) Tratamento de exceções no Save**):

* Exibir notificação de aviso (não de erro).
* Redirecionar para a lista, permitindo ao utilizador confirmar que os dados foram efetivamente salvos.

---

## 6) Resumo de boas práticas de Frontend

* Páginas de lista e edição devem ser claras, com títulos, ações visíveis e uso consistente de Material 3.
* Usar `RadzenStack` + `RadzenCard` para layout principal.
* `RadzenDataGrid` sempre em largura total.
* Evitar duplicação de lógica entre páginas — isolar ao máximo regras no serviço HTTP.
* Não usar variáveis com underscore; siga clean code também no code-behind.
* Chamar `StateHasChanged` ou `InvokeAsync(StateHasChanged)` em métodos assíncronos quando necessário para garantir atualização da UI.
