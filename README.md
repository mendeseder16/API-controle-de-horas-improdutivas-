# API-controle-de-horas-improdutivas-

Para um projeto de **Controle de Horas Improdutivas** em .NET, o ideal é seguir os princípios da **Clean Architecture** ou **Arquitetura em Camadas (N-Tier)**. Isso garante que sua lógica de cálculo de prejuízo não fique "misturada" com o código que recebe a requisição HTTP.

Abaixo, apresento uma estrutura simplificada, mas profissional, focada na separação entre **Comunicação (WebAPI)** e **Regra de Negócio (Domain/Services)**.

---

### 1. Estrutura do Projeto (Sugestão)
* **WasteWatch.API**: Controllers, Filtros e Configurações de Injeção de Dependência.
* **WasteWatch.Application**: DTOs (Data Transfer Objects), Interfaces e Serviços de Negócio.
* **WasteWatch.Domain**: Entidades e Regras de Negócio puras.
* **WasteWatch.Infrastructure**: Repositórios e Contexto do Banco de Dados (EF Core).

---

### 2. Camada de Domínio (Entidade)
Aqui definimos o que é uma "Atividade Improdutiva" e como ela se comporta.

```csharp
namespace WasteWatch.Domain.Entities;

public class TaskImpro
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string Descricao { get; set; } = string.Empty;
    public DateTime Inicio { get; set; }
    public DateTime? Fim { get; set; }
    public decimal ValorHoraUsuario { get; set; }

    // Regra de Negócio: Cálculo do prejuízo
    public decimal CalcularPrejuizo()
    {
        if (!Fim.HasValue) return 0;
        var duracaoSessao = Fim.Value - Inicio;
        return (decimal)duracaoSessao.TotalHours * ValorHoraUsuario;
    }
}
```

---

### 3. Camada de Regras de Negócio (Service)
Esta camada é o coração da aplicação. Ela não sabe se os dados vêm de uma API ou de um App Console.

```csharp
namespace WasteWatch.Application.Services;

public interface ITaskService
{
    TaskImpro CriarAtividade(string descricao, decimal valorHora);
    void FinalizarAtividade(Guid id);
    IEnumerable<TaskImpro> ListarTodas();
}

public class TaskService : ITaskService
{
    // Em um cenário real, injetaríamos o Repositório aqui
    private static readonly List<TaskImpro> _cache = new();

    public TaskImpro CriarAtividade(string descricao, decimal valorHora)
    {
        var novaTask = new TaskImpro { 
            Descricao = descricao, 
            Inicio = DateTime.Now, 
            ValorHoraUsuario = valorHora 
        };
        _cache.Add(novaTask);
        return novaTask;
    }

    public void FinalizarAtividade(Guid id)
    {
        var task = _cache.FirstOrDefault(t => t.Id == id);
        if (task != null) task.Fim = DateTime.Now;
    }

    public IEnumerable<TaskImpro> ListarTodas() => _cache;
}
```

---

### 4. Camada de Comunicação (Controller)
O Controller apenas recebe o input, chama o serviço e retorna o status HTTP correto.

```csharp
using Microsoft.AspNetCore.Mvc;
using WasteWatch.Application.Services;

namespace WasteWatch.API.Controllers;

[ApiController]
[Route("api/[controller]")]
public class TasksController : ControllerBase
{
    private readonly ITaskService _taskService;

    public TasksController(ITaskService taskService)
    {
        _taskService = taskService;
    }

    [HttpPost]
    public IActionResult Start([FromBody] string descricao, [FromHeader] decimal valorHora)
    {
        var task = _taskService.CriarAtividade(descricao, valorHora);
        return CreatedAtAction(nameof(GetAll), new { id = task.Id }, task);
    }

    [HttpPut("{id}/stop")]
    public IActionResult Stop(Guid id)
    {
        _taskService.FinalizarAtividade(id);
        return NoContent();
    }

    [HttpGet]
    public IActionResult GetAll()
    {
        var tasks = _taskService.ListarTodas().Select(t => new {
            t.Id,
            t.Descricao,
            t.Inicio,
            t.Fim,
            Prejuizo = t.CalcularPrejuizo()
        });
        return Ok(tasks);
    }
}
```

---

### Boas Práticas Aplicadas:
1.  **Injeção de Dependência**: O Controller não dá um `new TaskService()`. Ele recebe a interface via construtor, facilitando testes unitários.
2.  **Separação de Preocupações**: O Controller cuida do protocolo HTTP. O Service cuida do fluxo. A Entidade cuida do cálculo matemático.
3.  **Encapsulamento**: O cálculo do prejuízo está dentro da entidade `TaskImpro`, centralizando a lógica matemática em um único lugar.
