# Prompts del proyecto

## Pipeline de GitHub Actions

Los prompts utilizados para generar el pipeline están documentados en [prompts-crm.md](./prompts-crm.md), que incluye:

1. **Tests de backend** - Job para ejecutar tests con Jest
2. **Generación del build** - Job para compilar y generar artefactos
3. **Despliegue en EC2** - Job para desplegar en servidor EC2 vía SSH/rsync
4. **Trigger** - Configuración pull_request para push a ramas con PR abierto
