# arellan-marketing-funnels — Captación y Retención de Clientes

Módulo de marketing digital para la Clínica Automotriz Arellan Hnos. Cubre la captación de nuevos clientes, el portal público de consulta de estado de vehículos y las campañas de reactivación de clientes inactivos.

**Fase de activación:** Fase 3 (escala). No es bloqueante para el MVP.

## Embudo de Captación Digital

```
CLIENTE POTENCIAL
    │
    ▼
Busca en Google "taller mecánico Surquillo"
    │ → SEO básico en cliente.arellan.pe
    ▼
Landing page (cliente.arellan.pe)
    │ → Servicios, precios orientativos, ubicación
    │ → CTA: "Agenda tu revisión"
    ▼
Formulario de contacto / WhatsApp
    │ → Mensaje a WhatsApp Business del taller
    │ → O formulario web → notificación a ADMIN
    ▼
Visita al taller → OT registrada en sistema
    │ → Cliente registrado en BD
    ▼
Post-servicio: WhatsApp de seguimiento
    │ → "¿Cómo está tu vehículo?"
    │ → Solicitud de reseña (Google Maps)
    ▼
CLIENTE RECURRENTE
```

## Portal Público del Cliente (cliente.arellan.pe)

El portal público es accesible sin login para consulta básica:

### Consulta Anónima por Placa

```
GET /status/{plate}
→ Devuelve estado actual de la OT activa del vehículo (si existe)
→ Sin datos personales del cliente (solo placa, estado, descripción del trabajo)
```

| Estado visible | Descripción para el cliente |
|---------------|----------------------------|
| Ingresado | Tu vehículo ha sido recibido y está en cola de atención |
| En diagnóstico | Nuestro técnico está evaluando tu vehículo |
| Presupuestado | Tenemos un presupuesto listo para ti — te contactaremos |
| En proceso | Trabajo en progreso, tiempo estimado: X horas |
| Listo | ¡Tu vehículo está listo para ser recogido! |
| Entregado | Vehículo entregado |

### Registro Opcional de Cliente

Clientes pueden registrarse para acceso adicional:
- Historial completo de vehículos y servicios
- Notificaciones push de avance de OT
- Descarga de proformas y comprobantes

**Prerequisito:** Registro ante MINJUS (Ley 29733) requerido antes de activar este módulo.

## Campañas de Retención

### WhatsApp Business API (Fase 3)

```typescript
// Tipos de mensajes automatizados
interface WhatsAppCampaign {
  type: 'MAINTENANCE_REMINDER' | 'INACTIVE_REACTIVATION' | 'OT_UPDATE' | 'REVIEW_REQUEST'
}

// Ejemplo: recordatorio de mantenimiento preventivo
// Trigger: OT de "cambio de aceite" entregada hace 90 días
const maintenanceReminder = {
  type: 'MAINTENANCE_REMINDER',
  trigger: {
    service: 'CAMBIO_ACEITE',
    daysSinceDelivery: 90,  // ~3,000 km si viaja 1,000 km/mes
  },
  template: 'Tu {vehiculo} ya tiene {dias_desde_servicio} días desde el último cambio de aceite. ¿Agendamos?',
  cta: 'https://wa.me/51XXXXXXXXX?text=Quiero+agendar+mantenimiento',
}
```

### Segmentos de Clientes

| Segmento | Criterio | Acción |
|---------|---------|--------|
| VIP | >3 visitas/año o >S/.2,000 facturado | Atención prioritaria, descuento fidelidad |
| Activo | Visitó en los últimos 6 meses | Recordatorios de mantenimiento |
| En riesgo | 6-12 meses sin visita | Campaña de reactivación |
| Perdido | >12 meses sin visita | Campaña especial de recuperación |

## Gestión de Reseñas

### Google Maps

El sistema genera automáticamente una solicitud de reseña 24 horas después de cada entrega:

```
WhatsApp: "Hola {nombre}, esperamos que tu {vehiculo} esté andando de 10 🔧
¿Podrías dejarnos una reseña en Google? Solo toma 1 minuto:
[link Google Maps]"
```

**Regla:** Solo se envía si la OT fue marcada como "sin incidencias" por el mecánico y el cliente no reclamó durante el proceso.

### Monitoreo de Reputación

- Alerta a ADMIN si nueva reseña de 1-2 estrellas aparece en Google Maps
- Respuesta obligatoria dentro de 24 horas (guiada por el sistema con plantillas)

## SEO y Presencia Digital

### Palabras Clave Objetivo (Lima, Surquillo)

- "taller mecánico Surquillo"
- "cambio de aceite Surquillo Lima"
- "mecánico a domicilio Surquillo"
- "diagnóstico electrónico auto Lima Sur"
- "llave codificada Lima"

### Google My Business

Datos a mantener actualizados en el perfil GMB:
- Horarios actualizados (incluyendo feriados peruanos)
- Fotos del taller y equipo de trabajo
- Respuesta a todas las reseñas (buenas y malas)
- Publicaciones semanales de servicios destacados

## Variables de Entorno

```env
WHATSAPP_BUSINESS_TOKEN=<token>
WHATSAPP_PHONE_NUMBER_ID=<id>
WHATSAPP_VERIFY_TOKEN=<webhook-verify>
GOOGLE_MY_BUSINESS_API_KEY=<key>
MAINTENANCE_REMINDER_DAYS=90
INACTIVE_CUSTOMER_DAYS=180
```

## Compliance (Ley 29733)

Antes de enviar cualquier comunicación de marketing:
- Cliente debe haber dado consentimiento explícito (registro en portal)
- Opción de baja en cada mensaje ("Responde STOP para no recibir más mensajes")
- Registro de consentimientos almacenado en BD con fecha y método

El banco de datos de clientes debe estar registrado ante MINJUS antes de activar este módulo.
