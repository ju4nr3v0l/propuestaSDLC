# dotnet-security-validator - Instrucciones de Validación

## Base de Conocimiento

Este agente implementa las validaciones de seguridad de la **Guía Técnica de Seguridad Back .NET** de Sistecredito.

**Fuente principal:**
- https://docs.sistecreditocloud.com/ingenieria-de-software/calidad-en-la-codificacion/desarrollo-seguro/guia-segurirdad-back/guia-buenas-practicas-seguridad-back.html

**Documentos relacionados:**
- Práctica Back-to-Back con Azure AD B2C
- Aseguramiento Token JWT B2C
- Plantilla Clean Architecture .NET

---

## Regla 1: CSRF Protection (CRÍTICO)

### Descripción
**Item 4 de Guía de Seguridad:** Todos los endpoints HTTP deben estar protegidos contra CSRF usando Azure AD B2C o mTLS.

### Detección
Buscar controllers o endpoints que:
- NO tienen el atributo `[Authorize]`
- NO tienen el atributo `[AllowAnonymous]` con justificación documentada
- Son endpoints públicos sin protección

### Código Vulnerable
```csharp
[ApiController]
[Route("api/[controller]")]
public class PaymentController : ControllerBase
{
    // ❌ VULNERABLE: No tiene [Authorize]
    [HttpPost]
    public IActionResult ProcessPayment(PaymentRequest request)
    {
        return Ok();
    }
}
```

### Código Correcto
```csharp
[ApiController]
[Route("api/[controller]")]
[Authorize] // ✅ Protegido por Azure AD B2C
public class PaymentController : ControllerBase
{
    [HttpPost]
    public IActionResult ProcessPayment(PaymentRequest request)
    {
        return Ok();
    }
    
    // ✅ Excepción documentada para endpoints públicos
    [AllowAnonymous]
    [HttpGet("health")]
    public IActionResult Health() => Ok("healthy");
}
```

### Mensaje de Error
```
❌ CRÍTICO: CSRF Protection violación detectada

Archivo: PaymentController.cs
Línea: 15
Endpoint: POST /api/payment

Problema:
El endpoint ProcessPayment no tiene protección CSRF. Todos los endpoints deben usar [Authorize] con Azure AD B2C.

Fix Sugerido:
1. Agregar [Authorize] a nivel de controller o método
2. Si es endpoint público, usar [AllowAnonymous] y documentar justificación

Referencia:
https://docs.sistecreditocloud.com/.../guia-buenas-practicas-seguridad-back.html#item-4

Estado: ❌ PR bloqueado hasta corrección
```

---

## Regla 2: Error Handling (ALTO)

### Descripción
**Item 10 de Guía de Seguridad:** El manejo de errores debe hacerse mediante ExceptionFilter centralizado. No usar try-catch en controllers que revelen información sensible.

### Detección
Buscar en controllers:
- Bloques `try-catch` que retornan información de excepción
- `catch` que expone `ex.Message` o `ex.StackTrace` directamente
- Ausencia de GlobalExceptionFilter registrado

### Código Vulnerable
```csharp
[HttpPost]
public IActionResult CreateUser(UserRequest request)
{
    try 
    {
        _userService.Create(request);
        return Ok();
    }
    catch (Exception ex)
    {
        // ❌ VULNERABLE: Expone detalles internos
        return BadRequest(new { 
            error = ex.Message, 
            stack = ex.StackTrace 
        });
    }
}
```

### Código Correcto
```csharp
// En Startup.cs o Program.cs
services.AddControllers(options => 
{
    options.Filters.Add<GlobalExceptionFilter>(); // ✅
});

// Controller - No try-catch
[HttpPost]
public IActionResult CreateUser(UserRequest request)
{
    // ✅ Las excepciones las maneja GlobalExceptionFilter
    _userService.Create(request);
    return Ok();
}

// GlobalExceptionFilter.cs
public class GlobalExceptionFilter : IExceptionFilter
{
    private readonly ILogger<GlobalExceptionFilter> _logger;
    
    public void OnException(ExceptionContext context)
    {
        _logger.LogError(context.Exception, "Error processing request");
        
        // ✅ Respuesta segura sin detalles internos
        context.Result = new ObjectResult(new 
        {
            error = "Ha ocurrido un error procesando su solicitud",
            traceId = Activity.Current?.Id
        })
        {
            StatusCode = 500
        };
    }
}
```

---

## Regla 3: Input Validation (CRÍTICO)

### Descripción
**Item 4 de Guía de Seguridad:** Todos los DTOs de entrada deben validarse usando FluentValidation para prevenir inyecciones y datos maliciosos.

### Código Vulnerable
```csharp
public class CreateUserRequest
{
    // ❌ Sin validación
    public string Email { get; set; }
    public string Password { get; set; }
    public int Age { get; set; }
}

[HttpPost]
public IActionResult Create(CreateUserRequest request)
{
    // ❌ Sin validación explícita
    _userService.Create(request);
    return Ok();
}
```

### Código Correcto
```csharp
// ✅ Validador FluentValidation
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email requerido")
            .EmailAddress().WithMessage("Email inválido")
            .MaximumLength(256);
            
        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8)
            .Matches(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)");
            
        RuleFor(x => x.Age)
            .InclusiveBetween(18, 120);
    }
}

// Startup.cs
services.AddFluentValidationAutoValidation();
services.AddValidatorsFromAssemblyContaining<CreateUserRequestValidator>();
```

---

## Regla 4: SQL Injection Prevention (CRÍTICO)

### Descripción
**Item 4 de Guía de Seguridad:** Usar Entity Framework Core (ORM) para todas las operaciones de base de datos. Prohibido concatenar SQL manualmente.

### Código Vulnerable
```csharp
// ❌ VULNERABLE: SQL Injection
public User GetUser(string email)
{
    var sql = "SELECT * FROM Users WHERE Email = '" + email + "'";
    var user = _context.Users.FromSqlRaw(sql).FirstOrDefault();
    return user;
}
```

### Código Correcto
```csharp
// ✅ Entity Framework con LINQ
public User GetUser(string email)
{
    return _context.Users
        .Where(u => u.Email == email)
        .FirstOrDefault();
}

// ✅ Si necesitas SQL raw, usa parámetros
public User GetUserRaw(string email)
{
    return _context.Users
        .FromSqlRaw("SELECT * FROM Users WHERE Email = {0}", email)
        .FirstOrDefault();
}
```

---

## Regla 5: Logging Security (MEDIO)

### Descripción
No registrar información sensible en logs (passwords, tokens, datos personales). Usar ILogger correctamente.

### Código Vulnerable
```csharp
public IActionResult Login(LoginRequest request)
{
    // ❌ Password en logs
    Console.WriteLine($"Login attempt: {request.Email} - {request.Password}");
    
    var token = _authService.Authenticate(request);
    
    // ❌ Token en logs
    _logger.LogInformation($"Token generated: {token}");
    
    return Ok(token);
}
```

### Código Correcto
```csharp
public IActionResult Login(LoginRequest request)
{
    // ✅ Solo información no sensible
    _logger.LogInformation("Login attempt for user: {Email}", request.Email);
    
    var token = _authService.Authenticate(request);
    
    // ✅ No loguear el token completo
    _logger.LogInformation("Authentication successful for user: {Email}", request.Email);
    
    return Ok(token);
}
```

---

## Regla 6: HTTP Status Codes (BAJO)

### Descripción
Usar códigos HTTP correctos según RFC 7231 para mejorar interoperabilidad de APIs.

### Código con Mejora Posible
```csharp
// 💡 Puede mejorar
[HttpPost]
public IActionResult Create(UserRequest request)
{
    var user = _userService.Create(request);
    return Ok(user); // ❌ Debería ser 201 Created
}
```

### Código Correcto
```csharp
// ✅ Códigos HTTP correctos
[HttpPost]
public IActionResult Create(UserRequest request)
{
    var user = _userService.Create(request);
    return CreatedAtAction(nameof(Get), new { id = user.Id }, user);
}

[HttpPut("{id}")]
public IActionResult Update(int id, UserRequest request)
{
    _userService.Update(id, request);
    return NoContent();
}
```

---

## Formato de Salida JSON

```json
{
  "agent": "dotnet-security-validator",
  "version": "1.0.0",
  "timestamp": "2026-02-02T16:53:00Z",
  "summary": {
    "total_files_scanned": 45,
    "violations_found": 8,
    "critical": 3,
    "high": 2,
    "medium": 2,
    "low": 1
  },
  "violations": [
    {
      "rule": "csrf_protection",
      "severity": "CRITICAL",
      "file": "Controllers/PaymentController.cs",
      "line": 15,
      "message": "El endpoint ProcessPayment no tiene protección CSRF",
      "suggestion": "Agregar [Authorize] a nivel de controller o método",
      "reference": "https://docs.sistecreditocloud.com/.../guia-buenas-practicas-seguridad-back.html#item-4"
    }
  ],
  "blocking": true,
  "allow_merge": false
}
```

---

## Integración con Azure DevOps

### Pipeline YAML

```yaml
trigger:
  branches:
    include:
      - main
      - develop
      - feature/*

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: UseDotNet@2
  inputs:
    version: '8.x'

- script: |
    dotnet build
  displayName: 'Build'

- script: |
    copilot agent run dotnet-security-validator --path $(Build.SourcesDirectory)
  displayName: 'Security Validation'

- task: PublishBuildArtifacts@1
  condition: failed()
  inputs:
    pathToPublish: 'validation-report.json'
    artifactName: 'SecurityReport'
```

---

## Niveles de Enforcement

| Severidad | Bloquea PR | Bloquea Deploy | Notificación |
|-----------|------------|----------------|--------------|
| CRITICAL  | ✅ Sí      | ✅ Sí          | Email + Teams |
| HIGH      | ✅ Sí      | ✅ Sí          | Teams |
| MEDIUM    | ⚠️ Warning | ❌ No          | Teams |
| LOW       | 💡 Info    | ❌ No          | Ninguna |

---

## Actualización del Agente

```bash
# Sincronización automática cada hora
copilot agent sync

# O manualmente
copilot agent update dotnet-security-validator
```

Versión actual: **1.0.0-prototype**
