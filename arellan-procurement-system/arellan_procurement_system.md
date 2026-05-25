# arellan-procurement-system — Sistema de Compras e Importaciones

Módulo de gestión de compras, proveedores e importaciones con trazabilidad completa de costos y control de comisiones no declaradas.

**Contexto crítico:** Uno de los fraudes identificados fue que el empleado encargado de importaciones cobraba comisiones del 20-30% de los proveedores extranjeros sin declararlas. Este módulo cierra ese riesgo (R-BIZ-02, nivel 12 — Alto).

## Problema que Resuelve

```
SITUACIÓN ANTERIOR (fraude activo):
  Proveedor EE.UU. → cobra $100 por repuesto
  Empleado negociaba en secreto → proveedor devolvía $20-30 "comisión"
  Taller pagaba $100, sin saber que el precio real podía ser $70-80
  
SITUACIÓN CON EL SISTEMA:
  Cotización ingresada al sistema → precio fijado antes del pedido
  OWNER aprueba precio explícitamente → sin posibilidad de modificación posterior
  Cualquier descuento o ajuste requiere justificación escrita y nueva aprobación
```

## Flujo de Compra Nacional

```
1. NECESIDAD identificada
   → Por mecánico (solicitud desde tablet) o
   → Por sistema (stock crítico automático)
   
2. SOLICITUD DE COMPRA creada
   → Ítem, cantidad, justificación
   → Enviada a ADMIN/FINANCE para aprobación
   
3. APROBACIÓN según monto
   → ≤ S/.100: Finance directo
   → S/.101-500: Admin
   → >S/.500: Owner push
   
4. COTIZACIÓN con proveedor
   → Registrada en sistema con precio, proveedor y fecha
   → No se puede modificar después de aprobada
   
5. ORDEN DE COMPRA emitida
   → PDF generado por sistema con número correlativo
   → Enviada al proveedor (email o WhatsApp)
   
6. RECEPCIÓN
   → Cotejada con OC original (proveedor, ítem, cantidad, precio)
   → Diferencia > 5%: alerta + nueva aprobación requerida
   
7. INGRESO A INVENTARIO
   → Automático al confirmar recepción
   → Precio promedio ponderado actualizado
```

## Flujo de Importación Internacional

```
1. IDENTIFICACIÓN de repuesto no disponible localmente
   
2. BÚSQUEDA de proveedor (EE.UU., China, Europa)
   → Proveedor registrado en sistema con datos fiscales
   → Precio CIF declarado explícitamente
   
3. VALIDACIÓN DE MARGEN
   → Sistema calcula: (precio_venta_historico - precio_CIF) / precio_venta_historico
   → Si margen implícito > 35%: requiere justificación escrita
   → Si margen < 5%: alerta de precio anormalmente alto (posible sobrecosto)
   
4. APROBACIÓN DEL OWNER
   → Con precio y proveedor fijos → no modificables post-aprobación
   
5. PAGO AL PROVEEDOR
   → Solo transferencia bancaria al proveedor registrado
   → Sin pagos en efectivo para importaciones
   → Comprobante de pago adjunto obligatorio
   
6. SEGUIMIENTO ADUANERO
   → Número de tracking + agente de aduana registrado
   → Costos adicionales (flete, aduana, IGV importación) registrados explícitamente
   
7. RECEPCIÓN Y VERIFICACIÓN
   → Cotejado con factura original del proveedor
   → Cualquier diferencia → investigación con audit_log
```

## Modelo de Datos

```typescript
// Tabla: purchase_orders
model PurchaseOrder {
  id              String   @id @default(uuid())
  type            PurchaseType  // NATIONAL | IMPORT
  status          POStatus
  requestedBy     String   // employeeId
  approvedBy      String?  // employeeId
  providerId      String
  items           PurchaseOrderItem[]
  totalAmount     Decimal  @db.Decimal(10,2)
  currency        String   @default("PEN")  // PEN | USD
  exchangeRate    Decimal? @db.Decimal(8,4)  // Si es USD → tipo de cambio del día
  justification   String?
  approvalNotes   String?
  invoiceUrl      String?  // URL al comprobante en S3
  createdAt       DateTime @default(now())
  approvedAt      DateTime?
  receivedAt      DateTime?
}

// Tabla: providers
model Provider {
  id              String   @id @default(uuid())
  name            String
  type            ProviderType  // NATIONAL | FOREIGN
  country         String   @default("PE")
  taxId           String?  // RUC (Perú) o Tax ID (exterior)
  contactName     String?
  contactPhone    String?
  contactEmail    String?
  bankAccount     String?  // Cifrado AES-256
  isActive        Boolean  @default(true)
  createdAt       DateTime @default(now())

  purchaseOrders  PurchaseOrder[]
}
```

## Control de Comisiones

```typescript
@Injectable()
export class MarginValidationService {
  async validateImportMargin(
    itemId: string,
    purchasePrice: number,
    currency: 'PEN' | 'USD',
  ): Promise<MarginValidationResult> {
    const priceInPen = currency === 'USD'
      ? purchasePrice * await this.getExchangeRate()
      : purchasePrice

    const historicalSalePrice = await this.getAverageSalePrice(itemId)
    if (!historicalSalePrice) {
      return { valid: true, requiresJustification: false, note: 'Sin historial de ventas' }
    }

    const impliedMargin = (historicalSalePrice - priceInPen) / historicalSalePrice

    if (impliedMargin > 0.35) {
      // Margen > 35% — posible sobrecosto por comisión oculta
      return {
        valid: false,
        requiresJustification: true,
        note: `Margen implícito ${(impliedMargin * 100).toFixed(1)}% supera el 35% permitido`,
        suggestedMaxPrice: historicalSalePrice * 0.65,
      }
    }

    if (impliedMargin < 0.05) {
      // Margen < 5% — precio anormalmente alto
      return {
        valid: false,
        requiresJustification: true,
        note: `Margen implícito ${(impliedMargin * 100).toFixed(1)}% — precio posiblemente inflado`,
      }
    }

    return { valid: true, requiresJustification: false, impliedMargin }
  }
}
```

## Integración SUNAT (Fase 2)

Para compras nacionales con factura, el sistema valida automáticamente:

```typescript
// Validar que el proveedor nacional tiene RUC activo y habido en SUNAT
async validateProviderSunat(ruc: string): Promise<SunatValidation> {
  const response = await this.sunatService.consultaRuc(ruc)
  return {
    valid: response.estado === 'ACTIVO' && response.condicion === 'HABIDO',
    name: response.razonSocial,
    status: response.estado,
    condition: response.condicion,
  }
}
```

## Reportes de Compras

| Reporte | Frecuencia | Destinatario |
|---------|------------|-------------|
| Compras del mes por proveedor | Mensual | OWNER + FINANCE |
| Top proveedores por monto | Mensual | OWNER |
| Importaciones con margen fuera de rango | Inmediato | OWNER |
| Órdenes de compra pendientes de recepción | Semanal | ADMIN |
| Cotizaciones vencidas sin OC | Semanal | ADMIN |
