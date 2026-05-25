# arellan-testing — Suite de Tests de Integración

Suite de tests de integración, tests unitarios críticos y tests de contrato para el ecosistema digital Arellan. Complementa `arellan-e2e-tests` con cobertura en la capa de dominio y servicios.

## Estrategia de Testing

```
         Pirámide de Tests — Arellan
         
              ┌─────────┐
              │  E2E    │  ← arellan-e2e-tests (Playwright)
              │  15%    │     Flujos completos de negocio
              ├─────────┤
              │ INTEGR. │  ← Este módulo (Jest + Testcontainers)
              │  35%    │     Servicios + DB + Redis + BullMQ
              ├─────────┤
              │ UNIT    │  ← Este módulo (Jest)
              │  50%    │     Guards, validadores, lógica de negocio
              └─────────┘
```

## Stack de Testing

| Herramienta | Uso |
|-------------|-----|
| **Jest** | Framework base para unit e integración |
| **@nestjs/testing** | Module testing utilities para NestJS |
| **Testcontainers** | Instancias reales de PostgreSQL y Redis para tests de integración |
| **Supertest** | HTTP assertions para endpoints |
| **jest-mock-extended** | Mocking con TypeScript type safety |

## Tests Unitarios Críticos

### Guard MFA

```typescript
describe('MfaRequiredGuard', () => {
  let guard: MfaRequiredGuard

  beforeEach(() => {
    guard = new MfaRequiredGuard()
  })

  it('should allow owner with MFA verified', () => {
    const ctx = createMockContext({ role: UserRole.OWNER, mfaVerified: true })
    expect(guard.canActivate(ctx)).toBe(true)
  })

  it('should block owner without MFA verified', () => {
    const ctx = createMockContext({ role: UserRole.OWNER, mfaVerified: false })
    expect(() => guard.canActivate(ctx)).toThrow(ForbiddenException)
  })

  it('should allow mechanic without MFA (not required)', () => {
    const ctx = createMockContext({ role: UserRole.MECHANIC, mfaVerified: false })
    expect(guard.canActivate(ctx)).toBe(true)
  })
})
```

### Validación de Monto de Gasto

```typescript
describe('ExpenseAuthorizationService.getRequiredApprovalLevel', () => {
  let service: ExpenseAuthorizationService

  it.each([
    [50, ApprovalLevel.FINANCE],
    [100, ApprovalLevel.FINANCE],
    [101, ApprovalLevel.ADMIN],
    [500, ApprovalLevel.ADMIN],
    [501, ApprovalLevel.OWNER],
    [2000, ApprovalLevel.OWNER],
    [2001, ApprovalLevel.DUAL_OWNER],  // Doble aprobación para >S/.2,000
  ])('amount %i → approval level %s', (amount, expectedLevel) => {
    expect(service.getRequiredApprovalLevel(amount)).toBe(expectedLevel)
  })
})
```

### Geofence Calculation

```typescript
describe('GeofenceService', () => {
  let service: GeofenceService

  it('should return inside when at taller location', () => {
    // Coordenadas exactas del taller
    expect(service.isInsideFence(-12.1095, -77.0282)).toBe(true)
  })

  it('should return inside at 100m from taller', () => {
    // 100m al norte del taller
    expect(service.isInsideFence(-12.1086, -77.0282)).toBe(true)
  })

  it('should return outside at 250m from taller', () => {
    // 250m al norte del taller
    expect(service.isInsideFence(-12.1072, -77.0282)).toBe(false)
  })
})
```

## Tests de Integración

### Cashbox Service + PostgreSQL

```typescript
describe('CashboxService (integration)', () => {
  let module: TestingModule
  let service: CashboxService
  let prisma: PrismaService

  beforeAll(async () => {
    // Testcontainers inicia PostgreSQL 15 real
    const pgContainer = await new PostgreSqlContainer('postgres:15-alpine').start()

    module = await Test.createTestingModule({
      imports: [
        PrismaModule.forRoot({ url: pgContainer.getConnectionUri() }),
        CashboxModule,
      ],
    }).compile()

    service = module.get(CashboxService)
    prisma = module.get(PrismaService)
    await runMigrations(pgContainer.getConnectionUri())
  })

  it('should detect discrepancy correctly', async () => {
    const session = await service.openCashbox(adminId, { initialAmount: 500 })
    await service.recordPayment(session.id, { amount: 150, method: 'EFECTIVO' })
    const result = await service.closeCashbox(session.id, { finalAmount: 600 })

    expect(result.discrepancy).toBe(-50)
    expect(result.hasAlert).toBe(true)
  })

  it('should log discrepancy to audit_log', async () => {
    // Verificar que el audit_log tiene el evento de diferencia de caja
    const auditEntry = await prisma.auditLog.findFirst({
      where: { action: 'CASHBOX_DISCREPANCY' },
      orderBy: { createdAt: 'desc' },
    })
    expect(auditEntry).toBeTruthy()
    expect(auditEntry.metadata).toMatchObject({ discrepancy: -50 })
  })
})
```

### ExpenseAuthorization Flow + BullMQ

```typescript
describe('Expense Authorization Flow (integration)', () => {
  it('should enqueue push notification when expense requires owner approval', async () => {
    const expense = await expenseService.createExpense(financeUserId, {
      amount: 750,
      description: 'Compra materiales',
      providerId: testProviderId,
    })

    expect(expense.status).toBe(ExpenseStatus.PENDING_OWNER)

    // Verificar que se encoló el job de notificación
    const jobs = await notificationQueue.getWaiting()
    const pushJob = jobs.find(j => j.data.type === 'EXPENSE_APPROVAL_REQUEST')
    expect(pushJob).toBeTruthy()
    expect(pushJob.data.expenseId).toBe(expense.id)
  })
})
```

## Tests de Inmutabilidad del Audit Log

```typescript
describe('Audit Log Immutability (DB-level)', () => {
  it('should reject DELETE via raw SQL', async () => {
    const log = await prisma.auditLog.findFirst()
    await expect(
      prisma.$executeRaw`DELETE FROM audit_logs WHERE id = ${log.id}`
    ).rejects.toThrow(/permission denied|rule/i)
  })

  it('should reject UPDATE via raw SQL', async () => {
    const log = await prisma.auditLog.findFirst()
    await expect(
      prisma.$executeRaw`UPDATE audit_logs SET action = 'TAMPERED' WHERE id = ${log.id}`
    ).rejects.toThrow(/permission denied|rule/i)
  })

  it('should reject DELETE via Prisma', async () => {
    const log = await prisma.auditLog.findFirst()
    await expect(
      prisma.auditLog.delete({ where: { id: log.id } })
    ).rejects.toThrow()
  })
})
```

## Configuración de Cobertura

```json
// jest.config.ts
{
  "coverageThreshold": {
    "global": {
      "branches": 80,
      "functions": 80,
      "lines": 80,
      "statements": 80
    },
    "./src/modules/finance/": {
      "branches": 90,
      "functions": 90,
      "lines": 90
    },
    "./src/modules/auth/": {
      "branches": 95,
      "functions": 95,
      "lines": 95
    }
  }
}
```

Los módulos de finance y auth tienen requerimientos de cobertura más altos por ser módulos de seguridad crítica.

## CI Integration

```yaml
# .github/workflows/test.yml
- name: Run unit tests
  run: npm run test:unit -- --coverage

- name: Run integration tests
  run: npm run test:integration
  env:
    # Testcontainers se encarga de levantar PostgreSQL y Redis
    TESTCONTAINERS_RYUK_DISABLED: true

- name: Upload coverage report
  uses: codecov/codecov-action@v3
  with:
    fail_ci_if_error: true
    threshold: 80
```
