# dotnet-security-validator - Prototipo de Agente

✅ **PROTOTIPO COMPLETO** creado en: `~/sistecredito-agent-prototype/`

## Archivos Creados
- `config.json` - Configuración del agente
- `instructions.md` - Reglas detalladas (ver abajo)
- `example-usage.md` - Ejemplo completo de uso
- `README.md` - Este archivo

## Validación Basada en SisteDocs

Este agente valida código .NET contra:
- Guía Técnica Back .NET (https://docs.sistecreditocloud.com/...)
- Práctica Back-to-Back con Azure AD B2C
- Aseguramiento Token JWT B2C

## 6 Reglas de Seguridad

✅ CRÍTICO: CSRF Protection (Azure AD B2C)
✅ CRÍTICO: Input Validation (FluentValidation)  
✅ CRÍTICO: SQL Injection Prevention (Entity Framework)
✅ ALTO: Error Handling (ExceptionFilter centralizado)
⚠️ MEDIO: Logging Security (ILogger sin datos sensibles)
💡 BAJO: HTTP Status Codes

## Próximos Pasos

1. Revisar archivos en ~/sistecredito-agent-prototype/
2. Validar con equipo de Ingeniería
3. Aprobar para implementación completa
