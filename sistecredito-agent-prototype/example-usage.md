# Ejemplo Completo de Uso - dotnet-security-validator

Este documento muestra un ejemplo real de cómo el agente `dotnet-security-validator` detecta y ayuda a corregir vulnerabilidades de seguridad.

---

## Escenario: API de Pagos

Un desarrollador está creando un nuevo endpoint para procesar pagos en la aplicación de SisteCredito.

---

## Código Vulnerable (Antes)

```csharp
using Microsoft.AspNetCore.Mvc;
using System;
using System.Data.SqlClient;

namespace SisteCredito.Payment.Api.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class PaymentController : ControllerBase
    {
        private readonly string _connectionString;

        public PaymentController(IConfiguration config)
        {
            _connectionString = config.GetConnectionString("DefaultConnection");
        }

        // ❌ Problema 1: Sin [Authorize] - CSRF vulnerable
        [HttpPost]
        public IActionResult ProcessPayment(PaymentRequest request)
        {
            try
            {
                // ❌ Problema 2: Sin validación de entrada
                Console.WriteLine($"Processing payment: {request.Amount} for {request.UserId}");
                
                // ❌ Problema 3: SQL Injection vulnerable
                using var connection = new SqlConnection(_connectionString);
                connection.Open();
                
                var sql = $"INSERT INTO Payments (UserId, Amount, CardNumber) " +
                         $"VALUES ({request.UserId}, {request.Amount}, '{request.CardNumber}')";
                
                var command = new SqlCommand(sql, connection);
                command.ExecuteNonQuery();

                // ❌ Problema 4: Console.WriteLine con datos sensibles
                Console.WriteLine($"Payment successful. Card: {request.CardNumber}");

                // ❌ Problema 5: HTTP 200 en lugar de 201 Created
                return Ok(new { message = "Payment processed" });
            }
            catch (Exception ex)
            {
                // ❌ Problema 6: Expone detalles internos de la excepción
                return BadRequest(new 
                { 
                    error = ex.Message, 
                    stackTrace = ex.StackTrace,
                    source = ex.Source
                });
            }
        }

        [HttpGet("{id}")]
        public IActionResult GetPayment(string userId)
        {
            try
            {
                // ❌ Problema 7: Más SQL injection
                using var connection = new SqlConnection(_connectionString);
                connection.Open();
                
                var sql = "SELECT * FROM Payments WHERE UserId = '" + userId + "'";
                var command = new SqlCommand(sql, connection);
                var reader = command.ExecuteReader();
                
                // Lógica de mapeo...
                return Ok();
            }
            catch (Exception ex)
            {
                return StatusCode(500, ex.Message);
            }
        }
    }

    public class PaymentRequest
    {
        public int UserId { get; set; }
        public decimal Amount { get; set; }
        public string CardNumber { get; set; }
        public string Cvv { get; set; }
    }
}
```

---

## Ejecución del Agente

```bash
# Desde la terminal del desarrollador
$ copilot agent run dotnet-security-validator --path ./src/SisteCredito.Payment.Api/

🔍 Analizando proyecto: SisteCredito.Payment.Api
📁 Archivos escaneados: 45
⏱️  Tiempo: 3.2s

❌ 7 violaciones encontradas (3 CRÍTICAS, 2 ALTAS, 2 MEDIAS)

⚠️  PR será BLOQUEADO hasta corrección de violaciones CRÍTICAS y ALTAS
```

---

## Reporte JSON Generado

```json
{
  "agent": "dotnet-security-validator",
  "version": "1.0.0-prototype",
  "timestamp": "2026-02-02T16:45:32Z",
  "project": "SisteCredito.Payment.Api",
  "repository": "sistecredito/payment-api",
  "branch": "feature/add-payment-endpoint",
  "commit": "a1b2c3d4",
  
  "summary": {
    "total_files_scanned": 45,
    "violations_found": 7,
    "critical": 3,
    "high": 2,
    "medium": 2,
    "low": 0
  },
  
  "violations": [
    {
      "id": "CSRF001",
      "rule": "csrf_protection",
      "severity": "CRITICAL",
      "file": "Controllers/PaymentController.cs",
      "line": 18,
      "method": "ProcessPayment",
      "endpoint": "POST /api/payment",
      "message": "El endpoint ProcessPayment no tiene protección CSRF. Todos los endpoints deben usar [Authorize] con Azure AD B2C.",
      "code_snippet": "public IActionResult ProcessPayment(PaymentRequest request)",
      "suggestion": "Agregar [Authorize] a nivel de controller",
      "reference": "https://docs.sistecreditocloud.com/.../guia-buenas-practicas-seguridad-back.html#item-4",
      "blocking": true
    },
    {
      "id": "SQL001",
      "rule": "sql_injection",
      "severity": "CRITICAL",
      "file": "Controllers/PaymentController.cs",
      "line": 28,
      "method": "ProcessPayment",
      "message": "SQL Injection detectado: concatenación de variables en query SQL",
      "attack_example": "request.CardNumber = \"1234'; DROP TABLE Payments; --\"",
      "suggestion": "Usar Entity Framework Core",
      "reference": "https://docs.sistecreditocloud.com/.../guia-buenas-practicas-seguridad-back.html#item-4",
      "blocking": true
    },
    {
      "id": "SQL002",
      "rule": "sql_injection",
      "severity": "CRITICAL",
      "file": "Controllers/PaymentController.cs",
      "line": 48,
      "method": "GetPayment",
      "message": "SQL Injection detectado: concatenación de parámetro 'userId'",
      "attack_example": "userId = \"' OR '1'='1' --\"",
      "suggestion": "Usar Entity Framework con LINQ",
      "reference": "https://docs.sistecreditocloud.com/.../guia-buenas-practicas-seguridad-back.html#item-4",
      "blocking": true
    },
    {
      "id": "ERR001",
      "rule": "error_handling",
      "severity": "HIGH",
      "file": "Controllers/PaymentController.cs",
      "line": 38,
      "method": "ProcessPayment",
      "message": "Error handling inseguro: se exponen detalles de la excepción",
      "risk": "Revela información interna del sistema a atacantes",
      "suggestion": "Implementar GlobalExceptionFilter",
      "reference": "https://docs.sistecreditocloud.com/.../guia-buenas-practicas-seguridad-back.html#item-10",
      "blocking": true
    },
    {
      "id": "VAL001",
      "rule": "input_validation",
      "severity": "HIGH",
      "file": "Controllers/PaymentController.cs",
      "line": 80,
      "dto": "PaymentRequest",
      "message": "El DTO PaymentRequest no tiene validador FluentValidation",
      "suggestion": "Crear PaymentRequestValidator : AbstractValidator<PaymentRequest>",
      "reference": "https://docs.sistecreditocloud.com/.../guia-buenas-practicas-seguridad-back.html#item-4",
      "blocking": true
    },
    {
      "id": "LOG001",
      "rule": "logging_security",
      "severity": "MEDIUM",
      "file": "Controllers/PaymentController.cs",
      "line": 24,
      "message": "Se está logueando información sensible: 'CardNumber'",
      "risk": "Datos de tarjetas no deben estar en logs",
      "suggestion": "Usar ILogger y loguear solo últimos 4 dígitos",
      "blocking": false
    },
    {
      "id": "LOG002",
      "rule": "logging_security",
      "severity": "MEDIUM",
      "file": "Controllers/PaymentController.cs",
      "line": 24,
      "message": "Uso de Console.WriteLine en lugar de ILogger",
      "suggestion": "Inyectar ILogger<PaymentController>",
      "blocking": false
    }
  ],
  
  "enforcement": {
    "blocking": true,
    "allow_merge": false,
    "required_approvals": ["arquitecto-seguridad", "tech-lead"],
    "message": "Este PR tiene 5 violaciones bloqueantes que deben corregirse antes del merge."
  },
  
  "metadata": {
    "scan_duration_seconds": 3.2,
    "agent_version": "1.0.0-prototype",
    "knowledge_base_version": "2026.02.01"
  }
}
```

---

## Código Corregido (Después)

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;
using FluentValidation;
using SisteCredito.Payment.Api.Data;

namespace SisteCredito.Payment.Api.Controllers
{
    [Authorize] // ✅ Fix 1: Protección CSRF con Azure AD B2C
    [ApiController]
    [Route("api/[controller]")]
    public class PaymentController : ControllerBase
    {
        private readonly PaymentDbContext _context;
        private readonly ILogger<PaymentController> _logger;

        public PaymentController(
            PaymentDbContext context, 
            ILogger<PaymentController> logger)
        {
            _context = context;
            _logger = logger;
        }

        // ✅ Fix 2: HTTP 201 Created
        [HttpPost]
        public IActionResult ProcessPayment(PaymentRequest request)
        {
            // ✅ Fix 3: ModelState validado por FluentValidation
            
            // ✅ Fix 4: ILogger sin datos sensibles
            _logger.LogInformation(
                "Processing payment for user: {UserId}, Amount: {Amount}, Card: ****{LastFour}", 
                request.UserId, 
                request.Amount,
                request.CardNumber.Substring(request.CardNumber.Length - 4)
            );
            
            // ✅ Fix 5: Entity Framework previene SQL Injection
            var payment = new Payment 
            {
                UserId = request.UserId,
                Amount = request.Amount,
                CardNumber = request.CardNumber,
                CreatedAt = DateTime.UtcNow
            };
            
            _context.Payments.Add(payment);
            _context.SaveChanges();

            return CreatedAtAction(nameof(GetPayment), new { id = payment.Id }, payment);
        }

        // ✅ Fix 6: Sin try-catch (GlobalExceptionFilter lo maneja)
        [HttpGet("{id}")]
        public IActionResult GetPayment(int id)
        {
            // ✅ Entity Framework con LINQ
            var payment = _context.Payments
                .Where(p => p.Id == id)
                .FirstOrDefault();
            
            if (payment == null)
                return NotFound();
                
            return Ok(payment);
        }
    }

    // ✅ Fix 7: Validador FluentValidation
    public class PaymentRequestValidator : AbstractValidator<PaymentRequest>
    {
        public PaymentRequestValidator()
        {
            RuleFor(x => x.UserId)
                .GreaterThan(0)
                .WithMessage("UserId debe ser mayor a 0");
                
            RuleFor(x => x.Amount)
                .GreaterThan(0)
                .LessThanOrEqualTo(1000000);
                
            RuleFor(x => x.CardNumber)
                .NotEmpty()
                .CreditCard();
                
            RuleFor(x => x.Cvv)
                .Length(3, 4)
                .Matches(@"^\d+$");
        }
    }
}
```

---

## Resultado Final

```bash
$ copilot agent run dotnet-security-validator --path ./src/SisteCredito.Payment.Api/

🔍 Analizando proyecto: SisteCredito.Payment.Api
📁 Archivos escaneados: 45
⏱️  Tiempo: 2.8s

✅ 0 violaciones encontradas

✅ PR puede mergearse - Todas las validaciones pasaron
```

---

## Beneficios Demostrados

| Métrica | Antes | Después |
|---------|-------|---------|
| **Vulnerabilidades CRÍTICAS** | 3 | 0 |
| **Código vulnerable a SQL Injection** | Sí | No |
| **Protección CSRF** | No | Sí |
| **Manejo de errores seguro** | No | Sí |
| **Validación de entrada** | No | Sí |
| **Logs seguros** | No | Sí |
| **Tiempo de detección** | Manual (días) | Automático (3s) |
| **Conformidad con SisteDocs** | 30% | 100% |

---

## Flujo Completo de Desarrollo

```
1. Desarrollador escribe código
   ↓
2. Git commit
   ↓
3. Pre-commit hook ejecuta agente (local)
   ↓
4. Si pasa → push al repositorio
   Si falla → correcciones locales
   ↓
5. Crea Pull Request
   ↓
6. Azure Pipeline ejecuta agente (CI)
   ↓
7. Si pasa → Tech Lead puede aprobar
   Si falla → PR bloqueado
   ↓
8. Merge a develop/main
```

---

## Métricas de Adopción (Proyectadas)

Después de 3 meses de uso:

- **Reducción de vulnerabilidades en producción:** 85%
- **Tiempo de code review reducido:** 40%
- **Conformidad con estándares:** De 45% a 95%
- **Incidentes de seguridad:** De 12/mes a 2/mes

---

## Comandos Útiles

```bash
# Ejecutar localmente
copilot agent run dotnet-security-validator

# Solo archivos cambiados
copilot agent run dotnet-security-validator --changed-only

# Generar reporte HTML
copilot agent run dotnet-security-validator --output html

# Ver versión
copilot agent version dotnet-security-validator

# Actualizar
copilot agent update dotnet-security-validator
```

---

## Soporte

- **Documentación:** https://docs.sistecreditocloud.com
- **Teams:** #equipo-arquitectura
- **Email:** arquitectura@sistecredito.com

---

*Este ejemplo demuestra cómo el agente detecta 7 problemas críticos en 3 segundos.*
