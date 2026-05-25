# arellan-sandbox-labs — Laboratorio de Experimentación

Entorno de pruebas, prototipado y experimentación técnica para el ecosistema digital Arellan. Permite evaluar nuevas tecnologías, integraciones y features antes de incorporarlos al código de producción.

## Propósito

El sandbox tiene tres funciones:

1. **Evaluación de integraciones:** Probar APIs de terceros (SUNAT, Culqi, WhatsApp Business, ZKTeco) antes de implementarlas en producción
2. **Prototipado rápido:** Validar ideas técnicas sin arriesgar el código del sistema principal
3. **Entrenamiento:** Ambiente de práctica para nuevos miembros del equipo técnico

## Experimentos Activos

### EXP-001: SUNAT OSE Integration Test

```typescript
// Validación de integración con Nubefact (OSE SUNAT)
// Objetivo: confirmar flujo completo de emisión de factura electrónica

const testInvoice = {
  tipo_de_comprobante: '01',  // 01=factura, 03=boleta
  serie: 'F001',
  numero: '00000001',
  cliente_tipo_de_documento: '6',  // 6=RUC
  cliente_numero_de_documento: '20100130492',
  cliente_denominacion: 'Cliente de Prueba SAC',
  productos: [{
    codigo: 'SRV-001',
    descripcion: 'Servicio de mantenimiento preventivo',
    unidad_de_medida: 'ZZ',
    cantidad: 1,
    valor_unitario: 100.00,
    precio_unitario: 118.00,
  }],
}

// Resultado esperado: XML firmado digitalmente + CDR de SUNAT
```

**Estado:** En evaluación. Proveedor seleccionado: Nubefact (S/.29/mes).

### EXP-002: WhatsApp Business API Template Messages

```
Templates a aprobar en Meta for Developers:
  1. OT_STATUS_UPDATE — "Tu vehículo {{1}} está listo para recoger"
  2. EXPENSE_APPROVAL_REQUEST — "Tienes una solicitud de gasto de S/.{{1}} pendiente"
  3. MAINTENANCE_REMINDER — "Recordatorio: tu {{1}} cumple {{2}} días desde el último servicio"
  4. CASHBOX_DAILY_REPORT — Reporte diario de caja para owners

Estado: Pendiente de aprobación de templates en Meta Business Manager
```

### EXP-003: Culqi Webhook Idempotency

```typescript
// Verificar manejo correcto de webhooks duplicados de Culqi
// Culqi puede enviar el mismo evento múltiples veces

const webhookHandler = async (event: CulqiWebhookEvent) => {
  // Idempotency key: ID del evento Culqi
  const existing = await redis.get(`culqi:event:${event.id}`)
  if (existing) {
    logger.info(`Webhook duplicado ignorado: ${event.id}`)
    return  // 200 OK pero sin procesamiento
  }

  await redis.setex(`culqi:event:${event.id}`, 86400, 'processed')
  await processPayment(event)
}

// Tests: enviar el mismo event ID 3 veces → solo debe procesarse 1 vez
```

**Estado:** Implementado y validado. Listo para producción.

### EXP-004: Offline Queue Performance Test

```typescript
// Validar rendimiento de IndexedDB queue en mechanic-ui bajo carga
// Escenario: 50 acciones encoladas sin conexión → reconexión → sincronización

const offlineActions = Array.from({ length: 50 }, (_, i) => ({
  type: 'UPDATE_ORDER_STATUS',
  payload: { orderId: `test-${i}`, status: 'EN_PROCESO' },
  timestamp: Date.now(),
}))

// Métrica objetivo: sincronización completa en < 30 segundos
// Estado: PASSED — promedio 8 segundos para 50 acciones
```

**Estado:** Validado. Máximo confirmado: 8 horas de trabajo offline (capacidad de 480+ acciones).

### EXP-005: React Native Expo (Fase 3 Evaluation)

```typescript
// Evaluación de migración de arellan-mobile-app de PWA a React Native Expo
// Motivación: funcionalidades nativas (Face ID, notificaciones más confiables)
// 
// Resultado actual:
//   PRO: Push más confiable en iOS, Face ID/Touch ID nativo
//   CON: App Store review (semanas), costo de cuenta Apple ($99/año)
//       Dos bases de código a mantener (iOS + Android)
//
// Decisión: PWA para MVP, evaluar RN Expo en Fase 3 según necesidad real
```

## Entorno del Sandbox

```yaml
# Completamente aislado de producción y staging
DATABASE_URL: postgresql://sandbox:sandbox@localhost:5432/arellan_sandbox
REDIS_URL: redis://localhost:6379/2
NODE_ENV: sandbox
SUNAT_OSE_ENVIRONMENT: demo  # SUNAT demo environment
CULQI_SECRET_KEY: sk_test_xxx  # Culqi test keys
WHATSAPP_TEST_NUMBER: +51XXXXXXXXX  # Número de prueba propio
```

## Política del Sandbox

- **Datos:** Solo datos ficticios generados con faker-js — nunca datos reales de clientes
- **Secretos:** Solo credenciales de entornos demo/test — nunca producción
- **Código:** El código del sandbox nunca va a `main` directamente — siempre pasa por code review
- **Limpieza:** El sandbox se resetea mensualmente (primer lunes de cada mes)

## Experimentos Completados → Producción

| Experimento | Resultado | Implementado en |
|-------------|-----------|----------------|
| ZKTeco ADMS bridge | Exitoso | `arellan-iot-hardware-bridge` |
| JWT RS256 + Passport | Exitoso | `arellan-auth-service` |
| BullMQ queues | Exitoso | `arellan-workers` |
| Offline IndexedDB sync | Exitoso | `arellan-mechanic-ui` |
| Culqi webhook idempotency | Exitoso | `arellan-integrations-hub` |
