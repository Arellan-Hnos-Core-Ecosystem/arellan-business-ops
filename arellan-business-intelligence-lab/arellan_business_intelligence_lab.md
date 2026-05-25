# arellan-business-intelligence-lab — Inteligencia de Negocio

Análisis estratégico del negocio, definición de KPIs, métricas de rentabilidad y dashboards de decisión para los propietarios de la Clínica Automotriz Arellan Hnos.

**Nota:** Este módulo define los requerimientos de negocio e interpretación de datos. La implementación técnica del pipeline y los dashboards se encuentra en `arellan-data-intelligence`.

## Audiencia

| Rol | Necesidad de Información |
|-----|--------------------------|
| Edgar (Owner) | Rentabilidad total, eficiencia operativa, estado del negocio |
| Juan (Owner) | Seguridad del sistema, compliance, estado de fraude |
| Ana (Admin) | Productividad del equipo, OTs del día, cobros pendientes |
| Hija (Finance) | Flujo de caja, gastos vs ingresos, conciliación mensual |

## Panel de Control — Owners (Dashboard Gerencial)

Accesible en `arellan-mobile-app` → sección "Mi Negocio"

### Métricas Diarias (en tiempo real)

- **Ingresos del día** — suma de pagos confirmados (vs día anterior)
- **OTs activas** — cuántas están en proceso ahora mismo
- **Caja actual** — efectivo + digital en caja
- **Alertas activas** — diferencias de caja, gastos pendientes, stock crítico

### Métricas Semanales

- **Ingresos semana** — gráfica de barras 7 días
- **Top 5 servicios** — por facturación
- **Mecánico más productivo** — OTs cerradas / tiempo total
- **Tasa de conversión** — OTs recibidas vs aceptadas vs completadas

### Métricas Mensuales

- **P&L simplificado** — Ingresos - Costos directos - Gastos operativos = Utilidad
- **Margen bruto por categoría** — mecánica, electricidad, llaves/cerrajería, importaciones
- **Rotación de inventario** — stock vendido / stock promedio
- **Clientes recurrentes** — % de OTs de clientes que ya vinieron antes

## KPIs de Rentabilidad

### Por Servicio

```sql
-- Rentabilidad por tipo de servicio (mes actual)
SELECT
  ot.service_category,
  COUNT(*) as total_ots,
  SUM(ot.total_amount) as ingresos,
  SUM(ot.parts_cost) as costo_repuestos,
  SUM(ot.labor_cost) as costo_mano_obra,
  SUM(ot.total_amount - ot.parts_cost - ot.labor_cost) as utilidad_bruta,
  ROUND(
    (SUM(ot.total_amount - ot.parts_cost - ot.labor_cost) / SUM(ot.total_amount)) * 100, 2
  ) as margen_pct
FROM work_orders ot
WHERE ot.status = 'ENTREGADO'
  AND ot.delivered_at >= date_trunc('month', now())
GROUP BY ot.service_category
ORDER BY utilidad_bruta DESC;
```

### Por Mecánico

```sql
-- Productividad y rentabilidad por mecánico
SELECT
  a.full_name as mecanico,
  COUNT(ot.id) as ots_completadas,
  SUM(ot.total_amount) as facturado,
  AVG(EXTRACT(EPOCH FROM (ot.delivered_at - ot.created_at))/3600) as horas_promedio_ot,
  SUM(ot.total_amount) / NULLIF(COUNT(ot.id), 0) as ticket_promedio
FROM work_orders ot
JOIN accounts a ON a.id = ot.assigned_mechanic_id
WHERE ot.status = 'ENTREGADO'
  AND ot.delivered_at >= now() - INTERVAL '30 days'
GROUP BY a.id, a.full_name
ORDER BY facturado DESC;
```

## Detección de Anomalías de Negocio

### Cobros Fuera del Sistema

El riesgo R-BIZ-01 (cobros por Yape personal) se detecta comparando:

```
Ingresos registrados en sistema (día X)
    vs.
Depósitos bancarios reales (día X)

Diferencia > 5% → Alerta automática a OWNER
```

Esta comparación se hace semanalmente cuando el owner registra el extracto bancario en el sistema.

### Comisiones en Importaciones

```
Precio de compra registrado en sistema (importación X)
    vs.
Precio de mercado del mismo producto (referencia SUNAT + cotizaciones)

Margen > 35% sin justificación → Bloqueado + requiere explicación escrita de OWNER
```

### Gastos Duplicados

```typescript
// Detectar gastos potencialmente duplicados
interface DuplicateExpenseCheck {
  providerId: string
  amount: number
  periodDays: number  // Buscar gastos similares en los últimos N días
}

// Alerta si mismo proveedor + monto similar (±10%) en los últimos 7 días
```

## Métricas de Clientes

### Retención

- **Cliente recurrente:** vino al menos 2 veces en los últimos 12 meses
- **Cliente en riesgo:** no ha vuelto en 6+ meses (candidato para reactivación)
- **Cliente VIP:** >3 visitas/año o facturación total >S/.2,000

### Valor del Cliente (LTV simplificado)

```
LTV = Promedio ticket × Frecuencia anual × Años de vida esperada (3 años)
```

## Reportes Estratégicos Mensuales

El sistema genera automáticamente el día 1 de cada mes y envía por email a OWNER:

1. **Reporte P&L mensual** — ingresos, costos, utilidad, comparativa vs mes anterior
2. **Reporte de eficiencia operativa** — OTs, tiempos promedio, productividad por mecánico
3. **Reporte de inventario** — rotación, items críticos, valor del inventario actual
4. **Reporte de compliance** — MFA adoption, intentos fallidos, alertas de seguridad
5. **Reporte de clientes** — nuevos vs recurrentes, ticket promedio, top clientes

## Roadmap BI

| Fase | Funcionalidad |
|------|--------------|
| Fase 1 MVP | KPIs en tiempo real, dashboard básico en mobile app |
| Fase 2 | Reportes mensuales automatizados, análisis de rentabilidad por servicio |
| Fase 3 | ML: predicción de demanda, scoring de riesgo, churn prediction de clientes |
