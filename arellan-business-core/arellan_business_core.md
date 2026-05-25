# arellan-business-core — Lógica de Negocio Central

Documentación de los procesos de negocio centrales del ecosistema digital Arellan: flujos operativos, reglas de negocio y decisiones que determinan cómo funciona la Clínica Automotriz Arellan Hnos digitalmente.

## Flujo Principal: Ciclo de Vida de una Orden de Trabajo (OT)

```
CLIENTE INGRESA VEHÍCULO
    │
    ▼
Recepción registra OT en sistema (tablet)
    │ → Fotos obligatorias (mín. 4 fotos) + hash SHA-256
    │ → Estado: RECIBIDO
    ▼
Diagnóstico del mecánico
    │ → Mecánico selecciona síntomas + trabajo a realizar
    │ → Sistema sugiere repuestos del inventario (Fase 2+: IA)
    │ → Estado: EN_DIAGNOSTICO
    ▼
Presupuesto enviado al cliente
    │ → WhatsApp + portal cliente (si registrado)
    │ → Cliente aprueba/rechaza
    │ → Estado: PRESUPUESTADO
    ▼
Ejecución del trabajo
    │ → Mecánico registra avance en tablet
    │ → Consumo de repuestos descuenta inventario automáticamente
    │ → Estado: EN_PROCESO
    ▼
Control de calidad
    │ → Supervisor/Admin revisa trabajo (Fase 2)
    │ → Estado: EN_REVISION
    ▼
Pago y entrega
    │ → QR dinámico Yape/Plin generado desde sistema (nunca personal)
    │ → Comprobante electrónico SUNAT (Fase 2)
    │ → Estado: LISTO
    ▼
Entrega física del vehículo
    │ → Fotos del estado de entrega
    │ → Firma digital del cliente (Fase 2)
    │ → Estado: ENTREGADO
```

## Reglas de Negocio Críticas

### 1. QR de Pago Obligatoriamente desde el Sistema

**Regla:** Ningún mecánico ni empleado puede mostrar un QR personal (Yape/Plin personal) a un cliente. Todo pago debe pasar por el QR dinámico generado desde la tablet del sistema.

**Implementación:**
- La tablet solo tiene acceso a `taller.arellan.pe`
- El QR se genera dinámicamente con el monto exacto de la OT
- El pago se registra automáticamente en `financial_transactions`
- El cierre de caja diario cruza los QR generados vs pagos recibidos

**Por qué:** Este fue el mecanismo de fraude principal — Ricardo desviaba pagos a su Yape personal.

### 2. Control de Caja Diario

```
APERTURA DE CAJA (inicio del día)
    │ → ADMIN registra monto inicial (soles en efectivo)
    ▼
OPERACIONES DEL DÍA
    │ → Cada pago recibido: registrado inmediatamente
    │ → Tipo: EFECTIVO | YAPE | PLIN | TRANSFERENCIA | TARJETA
    ▼
CIERRE DE CAJA (fin del día)
    │ → ADMIN declara monto físico final en caja
    │ → Sistema calcula: monto_inicial + ingresos - gastos = esperado
    │ → Diferencia > S/.10: alerta automática a OWNER
    │ → Diferencia > S/.50: alerta P1 (fraude potencial)
    ▼
CONCILIACIÓN AUTOMÁTICA
    │ → Sistema verifica: suma de QR emitidos = pagos registrados
    │ → Cualquier gap queda en audit_log para revisión
```

### 3. Gestión de Inventario por OT

Cuando una OT consume repuestos, el sistema debe:
1. Verificar stock disponible antes de confirmar el trabajo
2. Reservar los repuestos al iniciar la OT (estado: RESERVADO)
3. Descontar del inventario cuando la OT pasa a EN_PROCESO
4. Si el repuesto no existe: generar solicitud de compra automática

```typescript
// Regla: nunca quedarse sin stock de repuestos críticos
const CRITICAL_STOCK_ITEMS = [
  'aceite-motor-10w40',
  'filtro-aceite-generico',
  'pastillas-freno-delanteras',
  'bujias-ngk',
]

// Alerta cuando stock cae por debajo del mínimo configurado
```

### 4. Autorización de Gastos por Monto

| Monto | Autorizador | Método |
|-------|-------------|--------|
| ≤ S/.100 | FINANCE (hija de Edgar) | Aprobación directa en sistema |
| S/.101 – S/.500 | ADMIN (Ana) | Aprobación en sistema |
| > S/.500 | OWNER (Edgar o Juan) | Push en arellan-mobile-app, requiere confirmación |
| Cualquier gasto de capital > S/.2,000 | Ambos OWNERS | Doble aprobación |

**Implementación crítica:** El botón de pago de un gasto está deshabilitado hasta recibir la aprobación del nivel correspondiente. La aprobación genera un token de un solo uso que autoriza la transacción.

### 5. Segregación de Funciones Financieras

- El empleado que registra un gasto **no puede aprobarlo** (mismo rol incluido)
- El empleado que recibe un pago **no puede modificarlo** retroactivamente
- Solo OWNER puede ver el balance total de caja + finanzas combinadas
- FINANCE puede ver ingresos y egresos pero no puede modificar audit_log

## Procesos de Apertura y Cierre del Taller

### Apertura (7:00 AM — Lunes a Sábado)

1. Mecánico registra check-in biométrico en ZKTeco
2. Admin abre caja: registra monto inicial en efectivo
3. Sistema verifica: ¿hay OTs pendientes del día anterior?
4. Sistema genera reporte matutino: OTs en proceso, stock crítico, cobros pendientes

### Cierre (6:00 PM — Lunes a Sábado)

1. Admin cierra todas las OTs del día (LISTO o pasa a mañana)
2. Admin realiza cierre de caja: cuenta efectivo físico
3. Sistema calcula y muestra diferencia vs esperado
4. Admin confirma cierre → reporte de caja enviado automáticamente a OWNER por WhatsApp
5. Mecánicos registran check-out biométrico

## Módulo de Importaciones (Fase 2)

Proceso crítico para controlar las comisiones no declaradas:

```
Proveedor extranjero identificado (EE.UU./China/Europa)
    │
    ▼
Registro de cotización en sistema
    │ → Precio CIF declarado
    │ → Proveedor registrado con RUC/ID fiscal
    ▼
Validación de margen
    │ → Sistema calcula margen implícito vs precio de venta histórico
    │ → Si margen > 35%: alerta automática + requiere justificación escrita
    ▼
Aprobación de importación
    │ → OWNER aprueba el pedido con precio y proveedor fijos
    ▼
Registro de pago al proveedor
    │ → Solo se puede pagar al proveedor registrado
    │ → No se permiten "comisiones" sin registro explícito
    ▼
Recepción de mercancía
    │ → Cotejado con orden de compra original
    │ → Diferencias requieren justificación y aprobación OWNER
```

## KPIs Operativos (Monitoreo Diario)

| KPI | Fórmula | Alerta |
|-----|---------|--------|
| Tiempo promedio OT | `suma(entregado_at - recibido_at) / count(OTs)` | > 5 días |
| OTs en proceso > 3 días | `count(OTs en_proceso, created > 3d)` | > 3 OTs |
| Diferencia de caja acumulada mensual | `suma(diferencias diarias)` | > S/.200/mes |
| Stock crítico | `count(items con stock ≤ stock_minimo)` | > 5 items |
| Cobros pendientes | `suma(OTs en estado LISTO no cobradas)` | > S/.500 |
