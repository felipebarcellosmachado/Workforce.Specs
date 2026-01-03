# Prompt Principal — Workforce CRUD Completo para `<Entity>` (Blazor + Radzen + .NET)

Este é o **documento principal** do prompt.  
Ele define a persona, variáveis, regras gerais, formato da saída, critérios de aceite e orientações globais.

Os detalhes específicos de cada camada (Repository, API, Serviço HTTP e Frontend) foram extraídos para **quatro documentos auxiliares** em `.md`, descritos na secção [Componentes do Prompt](#5-componentes-do-prompt-arquitetura-em-4-md).

---

## 1) Persona

Atue como **expert em C#, ASP.NET Core (.NET 10), EF Core, Blazor WebAssembly, UX/UI** e **Radzen** (Material 3).  
Siga boas práticas de clean code e **não use nomes de variáveis iniciados com underscore**.

---

## 2) Variáveis (substituir o que está entre `<>`)

Estas variáveis são utilizadas em todos os documentos auxiliares:

* `<Category>` = **Core**
* `<Group>` = **LeaveManagement***
* `<Entity>` = **LeaveType**
* `<BaseEntity>` = **LeaveType**
* `<Enum>` = **Enums**
* `<Folder>` = **<Category>.<Group>.<BaseEntity>**
* `<Domain>` (classe de domínio) = **Workforce.Domain.<Folder>.Entity.<Entity>.cs**
* `<RepositoryFile>` = **Workforce.Business.<Folder>.Repository.<Entity>Repository.cs**
* `<ServiceFiles>` =  
  * **Workforce.Services.<Folder>.<Entity>Service.cs**  
  * **Workforce.Services.<Folder>.I<Entity>Service.cs**
* `<ControllerFile>` = **Workforce.Server.Controllers.<Folder>.<Entity>Controller.cs**
* `<ListPage>` = **Workforce.Client.Pages.<Folder>.Pages.<Entity>sPage.razor** (+ code-behind **.razor.cs**)
* `<EditPage>` = **Workforce.Client.Pages.<Folder>.Pages.<Entity>Page.razor** (+ code-behind **.razor.cs**)
* `<EnvIdExpr>` = **AppState.Session.Environment.Id**  ← *(use esta expressão única para EnvironmentId)*

---

## 3) Regras Gerais

Estas regras aplicam-se a **todas** as camadas (Business, API, Serviço HTTP e Frontend):

* Use **nomes assíncronos** com sufixo `Async`.
* Todas as operações assíncronas devem aceitar `CancellationToken ct = default`.
* Consultas de lista: use `AsNoTracking()`.
* **Se `<Domain>` tiver propriedade `EnvironmentId`** (int):
  * implemente `GetAllByEnvironmentIdAsync(environmentId, ct)` no repositório e no serviço HTTP;
  * exponha rotas específicas no controller;
  * a UI deve filtrar por `<EnvIdExpr>`.
* Se `<Entity>` **herdar de uma tabela não abstrata**, operações de escrita usam **transação EF Core**.
* Siga **Material 3** no Radzen e **não fixe tamanhos** (layout responsivo).
* Internacionalização: use `IStringLocalizer<SharedResources>` (ex.: `L["Qualification"]`) em texto visível.
* Em campos de data faça os devidos tratamentos para banco de dados **PostgreSQL** (tipos, fuso horário, `DateOnly` vs `DateTime`).
* Coloque enums em `<Enum>` se necessário.
* Para cada `[ForeignKey]` da `<Entity>` (com exceção de `EnvironmentId`), use dropdowns na UI.  
  Implemente a busca adequada no service se necessário.
* **Se `<Entity>` não estiver no banco de dados, crie uma migração, execute-a e atualize o banco.**

---

## 4) Saída Esperada (formato)

Sempre que o modelo for solicitado a gerar código com base neste prompt:

* Para cada arquivo de código, comece com:

  ```text
  /// FILE: <caminho/arquivo.exato>
  <bloco de código completo>
  ```

Gere código completo e compilável para cada arquivo listado nos documentos auxiliares.

---

## 5) Componentes do Prompt (Arquitetura em 4 .md)

O prompt foi decomposto em quatro documentos especializados, cada um focado numa camada específica:

### Business (Repository)

Responsável pela definição do `<Entity>Repository`, acesso a dados via EF Core, métodos CRUD, filtros por `EnvironmentId` e migrações de banco.

**Documento**: `Workforce.Specs/Prompts/Crud/Components/Business.md`

### Api (Controller)

Responsável pelo `<Entity>Controller`, rotas Web API, códigos de status, e registro do repositório e contexto no `Workforce.Server/Program.cs`.

**Documento**: `Workforce.Specs/Prompts/Crud/Components/Api.md`

### Serviço Http (Chamadas HTTP à API)

Responsável pelo `<Entity>Service` e `I<Entity>Service`, chamadas via `HttpClient`, tratamento de respostas HTTP e registro no `Workforce.Client/Program.cs`.

**Documento**: `Workforce.Specs/Prompts/Crud/Components/HttpService.md`

### Frontend (Pages, Views, Components, Layout, UX)

Responsável pelas páginas Blazor/Radzen (`<ListPage>`, `<EditPage>`), layout, componentes, validações, integração com `IStringLocalizer<SharedResources>`, navegação, `MainLayout` e diretrizes de UX/UI (Material 3).

**Documento**: `Workforce.Specs/Prompts/Crud/Components/Frontend.md`

### 5.1) Como usar os componentes

Ao pedir geração de código para uma camada específica:

* **Para o repositório**: use este documento principal + `Components/Business.md`.
* **Para o controller**: use este documento principal + `Components/Api.md`.
* **Para o serviço HTTP**: use este documento principal + `Components/HttpService.md`.
* **Para o frontend Blazor/Radzen**: use este documento principal + `Components/Frontend.md`.

O modelo deve sempre respeitar a persona, variáveis, regras gerais e formato de saída definidos neste documento, além das instruções específicas em cada `.md` auxiliar.

---

## 6) Critérios de Aceite

A implementação será considerada adequada se:

* A solução compilar sem erros.
* O banco de dados estiver atualizado com a nova tabela.
* A migração for criada e aplicada com sucesso.
* Não houver erros de rota (uso correto de `Created` em vez de `CreatedAtAction` quando indicado).
* Não houver erros RZ9996 (validadores Radzen fora de `RadzenFormField`).
* Houver tratamento correto de exceções HTTP em chamadas de API.
* Endpoints da API responderem conforme descrito (200, 201, 204, 400, 404, etc.).
* O repositório siga o padrão assíncrono com `CancellationToken`.
* O serviço HTTP implemente a interface corretamente e faça chamadas HTTP adequadas.
* O controller valide dados e retorne status codes apropriados.
* A página de lista carregue registros (filtrando por ambiente quando aplicável).
* A página de edição crie/atualize/apague corretamente.
* Navegação por duplo clique na lista funcione para abrir a página de edição.
* UI siga Material 3, com layout responsivo e sem tamanhos fixos.
* Internacionalização funcional em todos os textos visíveis (via `IStringLocalizer<SharedResources>`).
* Nenhuma variável privada comece com underscore.
* Código siga princípios de clean code e padrões de projeto adequados.
* Transações EF Core aplicadas quando necessário (operações de escrita em entidades derivadas de tabelas concretas).
* `AsNoTracking()` utilizado em todas as consultas de lista.
* Tratamento adequado de campos `DateTime`/`DateOnly` para PostgreSQL.

---

## 7) Observações Finais

* Caso `<Entity>` herde de uma tabela não abstrata, os métodos do repositório devem ser adaptados para fazer operações de escrita dentro de transações quando não forem consultas.
* Todas as operações de escrita (Insert, Update, Delete) devem usar transações quando indicado.
* Operações de leitura (GetById, GetAll) não precisam de transações.
* Deve haver tratamento adequado de campos `DateTime` para PostgreSQL (UTC, conversões, tipos corretos).
* O service no Client deve usar `HttpClient` para comunicar com a API.
* Páginas Blazor devem ser componentes reutilizáveis e testáveis.
* SEMPRE criar e aplicar migração após adicionar novo `DbSet` ao contexto.
* Verificar se a tabela foi criada corretamente no banco PostgreSQL antes de testar os endpoints.
* Ordem de execução recomendada (no desenvolvimento):  
  **Domínio → Repository → Controller → Service → UI → DbContext → Migração → Testes**.
* Use `Created()` em vez de `CreatedAtAction()` quando quiser evitar erros de rota do tipo "No route matches the supplied values".
* Validadores Radzen devem ficar sempre fora de `RadzenFormField` (ver instruções de Frontend).
* Trate exceções HTTP adequadamente no code-behind das páginas (ver instruções de Frontend e Serviço HTTP).

---

## 8) Teste Manual (Checklist)

Ao final, recomenda-se:

* Testar manualmente todos os endpoints com ferramentas como curl, Postman ou equivalente.
* Validar a navegação da UI (lista → edição → volta à lista).
* Verificar mensagens de validação e notificações de sucesso/erro.
* Confirmar que filtros por ambiente (`EnvironmentId`) estão a funcionar conforme `<EnvIdExpr>`.
* Confirmar que datas são persistidas e exibidas corretamente, considerando fuso horário e tipo de coluna no PostgreSQL.

Se desejar, podem ser adicionados, em documentos separados, exemplos de requests curl, casos de teste e checklists de UI mais detalhados.