# arellan-e2e-tests — Tests End-to-End

Suite de tests automatizados end-to-end con Playwright que verifica los flujos críticos del ecosistema digital Arellan desde la perspectiva del usuario final.

## Stack de Testing

| Herramienta | Uso |
|-------------|-----|
| **Playwright** | E2E browser automation (Chromium, Firefox, WebKit) |
| **@playwright/test** | Framework de tests con fixtures y expect |
| **faker-js** | Generación de datos de prueba |
| **Docker Compose** | Entorno E2E aislado (backend + DB + Redis) |

## Flujos Críticos Cubiertos

### Flujo 1: Ciclo Completo de OT (Golden Path)

```typescript
test.describe('Work Order Complete Cycle', () => {
  test('create OT → process → pay → deliver', async ({ page, adminUser, mechanicUser }) => {
    // 1. Admin crea OT
    await page.goto('/orders/new')
    await page.fill('[name="clientName"]', 'Carlos García')
    await page.fill('[name="plate"]', 'ABC-123')
    await page.selectOption('[name="serviceType"]', 'MANTENIMIENTO')
    await page.click('[data-testid="submit-order"]')
    const orderId = await page.getAttribute('[data-testid="order-id"]', 'data-value')

    // 2. Mecánico toma la OT (en mechanic-ui)
    await page.goto(`/mechanic/orders/${orderId}`)
    await page.click('[data-testid="start-work"]')
    await expect(page.locator('[data-testid="order-status"]')).toHaveText('EN_PROCESO')

    // 3. Mecánico completa el trabajo
    await page.click('[data-testid="complete-work"]')
    await expect(page.locator('[data-testid="order-status"]')).toHaveText('LISTO')

    // 4. Admin genera QR de cobro
    await page.goto(`/admin/orders/${orderId}/payment`)
    await expect(page.locator('[data-testid="payment-qr"]')).toBeVisible()

    // 5. Pago registrado → OT pasa a ENTREGADO
    // Simular webhook de pago desde pasarela
    await simulatePaymentWebhook(orderId, 'YAPE')
    await expect(page.locator('[data-testid="order-status"]')).toHaveText('ENTREGADO')
  })
})
```

### Flujo 2: Autorización de Gastos

```typescript
test.describe('Expense Authorization Flow', () => {
  test('expense > S/.500 requires owner push approval', async ({ page, financeUser, ownerUser }) => {
    // Finance crea gasto
    await loginAs(page, financeUser)
    await page.goto('/finance/expenses/new')
    await page.fill('[name="amount"]', '750')
    await page.fill('[name="description"]', 'Compra aceite por mayor')
    await page.click('[data-testid="submit-expense"]')

    // Verificar que el gasto está PENDIENTE (no se puede pagar directamente)
    await expect(page.locator('[data-testid="expense-status"]')).toHaveText('PENDING_APPROVAL')
    await expect(page.locator('[data-testid="pay-btn"]')).toBeDisabled()

    // Owner aprueba en mobile app (simulado)
    await approveExpenseAsMobile(ownerUser, page.url().split('/').pop())

    // Verificar que ahora sí se puede pagar
    await page.reload()
    await expect(page.locator('[data-testid="expense-status"]')).toHaveText('APPROVED')
    await expect(page.locator('[data-testid="pay-btn"]')).toBeEnabled()
  })

  test('requester cannot approve their own expense', async ({ page, financeUser }) => {
    await loginAs(page, financeUser)
    // Crear gasto como finance
    const expenseId = await createExpense(page, 200)
    // Intentar aprobar el mismo gasto (aunque finance no puede, verificar 403)
    const response = await page.request.post(`/api/v1/finance/expenses/${expenseId}/approve`)
    expect(response.status()).toBe(403)
  })
})
```

### Flujo 3: Control de Caja

```typescript
test.describe('Cashbox Management', () => {
  test('cashbox discrepancy triggers alert', async ({ page, adminUser, ownerUser }) => {
    // Admin abre caja con S/.500
    await loginAs(page, adminUser)
    await page.goto('/admin/cashbox/open')
    await page.fill('[name="initialAmount"]', '500')
    await page.click('[data-testid="open-cashbox"]')

    // Simular un ingreso de S/.150
    await recordPayment(page, { amount: 150, method: 'EFECTIVO' })

    // Cierre de caja con S/.600 (debería ser S/.650 → diferencia S/.50)
    await page.goto('/admin/cashbox/close')
    await page.fill('[name="finalAmount"]', '600')
    await page.click('[data-testid="close-cashbox"]')

    // Verificar que se muestra la diferencia
    await expect(page.locator('[data-testid="discrepancy"]')).toHaveText('-S/.50.00')
    await expect(page.locator('[data-testid="discrepancy-alert"]')).toBeVisible()

    // Verificar que el owner recibió notificación (mock del servicio de notificaciones)
    expect(notificationsMock.getLastNotification(ownerUser.id)).toMatchObject({
      type: 'CASHBOX_DISCREPANCY',
      amount: -50,
    })
  })
})
```

### Flujo 4: Acceso RBAC

```typescript
test.describe('RBAC Access Control', () => {
  test('mechanic cannot access finance module', async ({ page, mechanicUser }) => {
    await loginAs(page, mechanicUser)
    const response = await page.request.get('/api/v1/finance/summary')
    expect(response.status()).toBe(403)
  })

  test('finance role data is masked for mechanics', async ({ page, mechanicUser }) => {
    await loginAs(page, mechanicUser)
    const response = await page.request.get(`/api/v1/orders/${orderId}`)
    const data = await response.json()
    expect(data.client.phone).toBeUndefined()
    expect(data.client.email).toBeUndefined()
    expect(data.client.dni).toBeUndefined()
    // Solo debe ver placa y modelo
    expect(data.vehicle.plate).toBeDefined()
    expect(data.vehicle.model).toBeDefined()
  })

  test('MFA required for finance access', async ({ page, financeUserNoMfa }) => {
    await loginAs(page, financeUserNoMfa)
    const response = await page.request.get('/api/v1/finance/transactions')
    expect(response.status()).toBe(403)
    expect((await response.json()).code).toBe('MFA_REQUIRED')
  })
})
```

### Flujo 5: Inmutabilidad del Audit Log

```typescript
test.describe('Audit Log Immutability', () => {
  test('audit logs cannot be deleted', async ({ prisma, adminUser }) => {
    const logId = (await prisma.auditLog.findFirst()).id
    await expect(
      prisma.$executeRaw`DELETE FROM audit_logs WHERE id = ${logId}`
    ).rejects.toThrow()
  })

  test('audit logs cannot be updated', async ({ prisma }) => {
    const logId = (await prisma.auditLog.findFirst()).id
    await expect(
      prisma.$executeRaw`UPDATE audit_logs SET action = 'TAMPERED' WHERE id = ${logId}`
    ).rejects.toThrow()
  })
})
```

## Configuración

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './tests',
  fullyParallel: false,  // Tests de negocio son secuenciales (comparten estado)
  retries: process.env.CI ? 2 : 0,
  timeout: 30_000,
  globalSetup: './tests/setup/global.ts',
  globalTeardown: './tests/setup/teardown.ts',

  projects: [
    { name: 'admin-flows', use: { ...devices['Desktop Chrome'] } },
    { name: 'mechanic-flows', use: { ...devices['iPad Pro 11'] } },  // Touch device
    { name: 'mobile-app', use: { ...devices['iPhone 14'] } },
  ],

  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
})
```

## Entorno E2E

```yaml
# docker-compose.e2e.yml
services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: arellan_e2e
      POSTGRES_PASSWORD: testpass

  redis:
    image: redis:7-alpine

  api:
    build: ../../arellan-repos/arellan-platform/arellan-backend-api
    environment:
      DATABASE_URL: postgresql://postgres:testpass@db:5432/arellan_e2e
      REDIS_URL: redis://redis:6379
      NODE_ENV: test
    depends_on: [db, redis]
```

## Cobertura Mínima Requerida

| Módulo | Tests E2E Obligatorios |
|--------|----------------------|
| Órdenes de trabajo | Golden path completo |
| Caja y pagos | Apertura, cobro con QR, cierre, diferencia |
| Autorización de gastos | Cada nivel de monto |
| Control de acceso | Todos los roles intentando acceder a módulos no autorizados |
| Audit log | Inmutabilidad |

La suite E2E se ejecuta automáticamente en el pipeline CI/CD antes de cualquier merge a `main`.
