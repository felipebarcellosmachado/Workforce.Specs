# Workforce — Availability — Business (Repository & DB)

Este documento define as regras específicas para a **camada de Business** da entidade `<Entity>` (Availability):

* Repositório EF Core (`<Entity>Repository`).
* Acesso a dados, métodos CRUD e filtros por `EnvironmentId`.
* Atualização do `DbContext` e migrações de banco (PostgreSQL).

Respeite sempre:

* A persona, variáveis e regras gerais definidas em  
  `docs/Workforce.Availability.CRUD.Main.md`.
* O formato de saída descrito em **Saída Esperada (formato)** do documento principal.

---

## 1) Repositório

**Arquivo**: `<RepositoryFile>`  
**Classe**: `<Entity>Repository`

**Dependências injetadas**:  
Utilize o `DbContext` apropriado do projeto, por exemplo:

* `WorkforceDbContext` (nome coerente com o restante da solução).

### 1.1) Métodos públicos obrigatórios

Implemente os seguintes métodos assíncronos:

**a)** `Task<<Domain>?> GetByIdAsync(int id, CancellationToken ct = default)`  
* Retorna a entidade ou `null` se não encontrada.

**b)** `Task<IList<<Domain>>> GetAllAsync(CancellationToken ct = default)`  
* Retorna todas as entidades.
* Usa obrigatoriamente `AsNoTracking()` para otimizar consultas de lista.

**c)** *(Se existir `EnvironmentId` em `<Domain>`)*  
`Task<IList<<Domain>>> GetAllByEnvironmentIdAsync(int environmentId, CancellationToken ct = default)`  

* Filtra por `EnvironmentId`.
* Usa obrigatoriamente `AsNoTracking()`.

**d)** `Task<<Domain>> InsertAsync(<Domain> entity, CancellationToken ct = default)`  

* Insere e retorna a entidade inserida.
* Se `<Entity>` herdar de tabela **não abstrata**, execute a operação dentro de uma **transação EF Core**.

**e)** `Task<<Domain>?> UpdateAsync(<Domain> entity, CancellationToken ct = default)`  

* Carrega a entidade original via `GetByIdAsync`.
* Se não existir, retorna `null`.
* Atualiza apenas campos **não-chave**; deve preservar:
  * `Id`;
  * `EnvironmentId` se for considerado chave lógica/parte do escopo.
* Utilize uma *tracked entity* para update (não use `AsNoTracking()` aqui).
* Se `<Entity>` herdar de tabela concreta, execute o update dentro de uma **transação EF Core**.

**f)** `Task<bool> DeleteByIdAsync(int id, CancellationToken ct = default)`  

* Retorna `true` se removeu; `false` se não encontrou a entidade.
* Se `<Entity>` herdar de tabela concreta, use uma **transação** para o delete.

### 1.2) Observações específicas

* Adote `AsNoTracking()` em todos os métodos de lista:
  * `GetAllAsync`
  * `GetAllByEnvironmentIdAsync`
* Para update:
  * Trabalhe com entidades **rastreadas** (`tracked`) pelo contexto.
  * Considere padrão: carregar com `FindAsync`/`SingleOrDefaultAsync` sem `AsNoTracking()`, aplicar alterações nas propriedades e chamar `SaveChangesAsync`.
* Tratamento de datas para PostgreSQL:
  * `DateTime` armazenado como `timestamp with time zone` quando apropriado.
  * Se utilizar `DateOnly`, mapeie para `date`.
  * Considere uso de UTC quando fizer sentido para o domínio.

---

## 2) Atualizar DbContext e Criar Migração

Após definir a nova entidade e o repositório, deve-se garantir que o **banco PostgreSQL** esteja atualizado.

### 2.1) Atualizar o `WorkforceDbContext`

**Arquivo** (exemplo):  
`Workforce.Db/Workforce.Db/Db/WorkforceDbContext.cs`

Se a entidade ainda não estiver incluída no contexto, adicione um `DbSet`:

```csharp
public virtual DbSet<<Domain>> <Entities> { get; set; } = default!;
```

Exemplo de padrão:

```csharp
public virtual DbSet<Domain.Core.HumanResourceManagement.RiskFactor.Entity.RiskFactor> RiskFactors { get; set; } = default!;
```

Ajuste para `<Entity>`/`<Domain>` e `<Entities>` coerentes com Availability.

### 2.2) Criar a migração

Num terminal/PowerShell:

**Navegue até o projeto de banco (Workforce.Db):**

```powershell
Set-Location "C:\Users\use\source\vs\Business\Workforce\Workforce.Db\Workforce.Db"
```

**Crie a migração:**

```powershell
dotnet ef migrations add Add<Entity>Table --context WorkforceDbContext
```

Exemplo concreto:

```powershell
dotnet ef migrations add AddAvailabilityTable --context WorkforceDbContext
```

### 2.3) Aplicar a migração

Execute:

```powershell
dotnet ef database update --context WorkforceDbContext
```

### 2.4) Verificar a migração

Após o comando acima:

**Verificar arquivo de migração:**

* Localização típica:  
  `Workforce.Db/Workforce.Db/Migrations/<timestamp>_Add<Entity>Table.cs`
* Verifique colunas, constraints, FKs e índices.

**Verificar a tabela no PostgreSQL:**

* A tabela `<Entity>` deve existir com todas as colunas definidas em `<Domain>`.
* Foreign keys configuradas corretamente.
* `DateTime` como `timestamp with time zone` quando apropriado.
* `DateOnly` como `date`.
* Campos obrigatórios com `NOT NULL`.

**Saída esperada do database update:**

```
Build started...
Build succeeded.
Done.
```

### 2.5) Observações de migração

* Verifique se a entidade já não está no `WorkforceDbContextModelSnapshot.cs` antes de criar nova migração.
* Se a migração falhar:
  * Use `dotnet ef migrations remove` para removê-la antes de tentar novamente.
* Para listar migrações:

```powershell
dotnet ef migrations list --context WorkforceDbContext
```

* Para reverter a última migração:

```powershell
dotnet ef database update <NomeDaMigraçãoAnterior> --context WorkforceDbContext
dotnet ef migrations remove --context WorkforceDbContext
```

* Para gerar um script SQL sem aplicar diretamente:

```powershell
dotnet ef migrations script --context WorkforceDbContext --output migration.sql
```

* Em casos de herança TPT (Table-Per-Type), a migração criará tabelas separadas e as FKs apropriadas.

---

## 3) Notas de Business e Persistência

* Caso `<Entity>` herde de uma tabela não abstrata:
  * Operações de escrita (Insert, Update, Delete) devem usar transações EF Core.
  * Operações de leitura (GetById, GetAll) em geral não precisam de transações.
* Sempre sincronizar o modelo de domínio, o `DbContext` e o banco de dados via migrações.
* Garanta que o repositório inclua métodos que respeitem os filtros de ambiente (`EnvironmentId`) quando aplicável.