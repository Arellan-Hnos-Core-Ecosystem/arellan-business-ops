# arellan-business-ops — Operaciones de Negocio y Calidad

Repositorio macro que agrupa los módulos orientados a la operación estratégica del negocio: inteligencia comercial, compras, marketing, testing y entornos de prueba del ecosistema digital Arellan.

## Submódulos

| Submódulo | Descripción | Fase |
|-----------|-------------|------|
| `arellan-business-core` | Lógica de negocio central: procesos, reglas y flujos clave | Fase 1 MVP |
| `arellan-business-intelligence-lab` | Análisis de negocio, KPIs y reportes estratégicos | Fase 2 |
| `arellan-procurement-system` | Gestión de compras e importaciones con control de comisiones | Fase 2 |
| `arellan-marketing-funnels` | Captación de clientes, portal público, reseñas | Fase 3 |
| `arellan-e2e-tests` | Tests end-to-end con Playwright para flujos críticos | Fase 1+ |
| `arellan-testing` | Suite de tests de integración y contracts | Fase 1+ |
| `arellan-sandbox-labs` | Entorno de experimentación y pruebas de concepto | Ongoing |

## Prioridades por Fase

### Fase 1 — MVP (10-12 semanas)
- `arellan-business-core`: flujos de OT, caja, gastos
- `arellan-e2e-tests`: tests críticos de caja y autorización de gastos
- `arellan-testing`: tests de integración del backend

### Fase 2 — Consolidación (8-10 semanas)
- `arellan-procurement-system`: importaciones con auditoría de comisiones
- `arellan-business-intelligence-lab`: dashboards de rentabilidad

### Fase 3 — Escala (10-14 semanas)
- `arellan-marketing-funnels`: captación digital de nuevos clientes

## Acceso

Acceso restringido según módulo. Los datos de business intelligence son confidenciales y solo accesibles para OWNER y ADMIN.
