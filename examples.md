# 🧠 Backend Engineering Rules (NestJS)

## ❗ Error Handling

### Do not abuse `try/catch`

* Let the **global exception handler** handle errors
* Avoid `try/catch` unless:

  * adding context
  * translating external errors

#### ❌ Bad

```ts
try {
  await this.userService.create(data)
} catch (e) {
  // silently ignore
}
```

#### ❌ Bad

```ts
try {
  await this.repo.save(entity)
} catch (e) {
  return null
}
```

#### ✅ Good

```ts
await this.userService.create(data)
```

#### ✅ Good (with context)

```ts
try {
  await externalApi.call()
} catch (e) {
  this.logger.error('External API failed', { userId })
  throw new ExternalServiceException()
}
```

---

### BullMQ processors

* Do NOT wrap processors in `try/catch`
* Let BullMQ handle retries and failure states

#### ❌ Bad

```ts
@Processor('queue')
async handle(job: Job) {
  try {
    await this.service.process(job.data)
  } catch (e) {
    // breaks retry mechanism
  }
}
```

#### ✅ Good

```ts
@Processor('queue')
async handle(job: Job) {
  await this.service.process(job.data)
}
```

---

### Handle worker errors explicitly

Use `OnWorkerEvent`

#### ✅ Example

```ts
@OnWorkerEvent('failed')
onFailed(job: Job, err: Error) {
  this.logger.error('Job failed', {
    jobId: job.id,
    entityId: job.data.userId,
    error: err.message,
  })
}
```

---

### Never silently swallow errors

* No empty `catch`
* Always log or rethrow

---

## 🔁 Async Jobs & Idempotency

### Always use idempotent jobs

* Set deterministic `jobId`

#### ❌ Bad

```ts
await queue.add('send-email', { userId })
```

#### ✅ Good

```ts
await queue.add('send-email', { userId }, {
  jobId: `send-email:${userId}`,
})
```

---

### Always design failure recovery

Every async operation must define:

* retry strategy
* recovery strategy

#### Example strategies

* cron job retry
* manual reprocessing
* dead-letter queue

#### ✅ Example

```ts
// cron job that retries failed jobs
@Cron('*/5 * * * *')
async retryFailedJobs() {
  await this.jobService.retryFailed()
}
```

---

### Always implement background error handler

For fire-and-forget promises:

#### ❌ Bad

```ts
this.emailService.send(email)
```

#### ✅ Good

```ts
this.emailService.send(email).catch((e) => {
  this.logger.error('Email failed', { email })
})
```

---

## 🧾 Logging

### Always log entity IDs on error

#### ❌ Bad

```ts
this.logger.error('Failed to process')
```

#### ✅ Good

```ts
this.logger.error('Failed to process user', {
  userId,
  orderId,
})
```

---

## ✅ Validation

### Validate ALL incoming data

* params
* query
* body

### Rules

* trim strings
* enforce min/max
* validate arrays (length + uniqueness)

#### ❌ Bad

```ts
@Get(':id')
get(@Param('id') id: string) {}
```

#### ✅ Good

```ts
const GetUserParams = z.object({
  id: z.string().uuid(),
})
```

---

### Example DTO

```ts
const CreateUserDto = z.object({
  name: z.string().min(1).max(100).transform((v) => v.trim()),
  roles: z.array(z.string()).max(10).refine(
    (arr) => new Set(arr).size === arr.length,
    { message: 'Roles must be unique' },
  ),
})
```

---

## 🧩 Type Safety

### Use `satisfies never` in switch

#### ❌ Bad

```ts
switch (status) {
  case 'A': ...
  case 'B': ...
}
```

#### ✅ Good

```ts
switch (status) {
  case 'A': ...
  case 'B': ...
  default:
    const _exhaustive: never = status
    return _exhaustive
}
```

---

## 🏗️ Architecture

### Do not export services from domain modules

* prevents tight coupling

#### ❌ Bad

```ts
@Module({
  providers: [UserService],
  exports: [UserService],
})
```

---

### Use shared "domain lib" for cross-domain logic

#### ✅ Good

```ts
@Module({
  providers: [UserLibService],
  exports: [UserLibService],
})
```

---

### Separate domain vs infrastructure

#### Structure

```
/domain
  /user
  /patient
  /reservation

/infrastructure
  /auth
  /bullmq
  /graphql
  /dataloader
```

---

### Each domain = its own module

#### ❌ Bad

```
/events-module
  events
  recurring-events
```

#### ✅ Good

```
/events
/recurrent-events
```

---

## 📦 Code Structure

### Keep files < 1000 lines

* split large logic into multiple services

---

### Keep functions < 100 lines

#### ❌ Bad

```ts
async processEverything() {
  // 200 lines
}
```

#### ✅ Good

```ts
async process() {
  await this.validate()
  await this.calculate()
  await this.persist()
}
```

---

## ⚡ Performance & Concurrency

### Avoid `Promise.all` on DB operations

#### ❌ Bad

```ts
await Promise.all(users.map(u => this.repo.save(u)))
```

#### ✅ Good

```ts
await this.repo.insertMany(users)
```

---

### Use `p-limit` for concurrency control

#### ❌ Bad

```ts
await Promise.all(urls.map(fetch))
```

#### ✅ Good

```ts
const limit = pLimit(5)
await Promise.all(urls.map(url => limit(() => fetch(url))))
```

---

## 🔁 Jobs & Scaling

### Separate workers and cron jobs

* workers → scalable
* cron → single instance

---

## 🧱 API Design

### Use object for same-type params

#### ❌ Bad

```ts
updateUser(id: string, name: string, age: number)
```

#### ✅ Good

```ts
updateUser(input: { id: string; name: string; age: number })
```

---

## 💰 Numbers

### Avoid JS float calculations

#### ❌ Bad

```ts
0.1 + 0.2 === 0.30000000000000004
```

#### ✅ Good

```ts
import Decimal from 'decimal.js'

new Decimal(0.1).plus(0.2).toNumber()
```

### Be aware of division rounding

* Division can produce non-integer results — always decide whether to round, truncate, or keep full precision
* Make the rounding strategy explicit; never silently pass unrounded values to APIs, databases, or user-facing output

#### ❌ Bad

```ts
const pricePerItem = totalPrice / quantity
// 10 / 3 = 3.3333333333333335 — stored/displayed without rounding
await this.repo.save({ pricePerItem })
```

#### ❌ Bad

```ts
const percentage = (completed / total) * 100
return `${percentage}%` // "33.33333333333333%"
```

#### ✅ Good

```ts
const pricePerItem = new Decimal(totalPrice).div(quantity).toDecimalPlaces(2).toNumber()
await this.repo.save({ pricePerItem })
```

#### ✅ Good

```ts
const percentage = Math.round((completed / total) * 100)
return `${percentage}%` // "33%"
```

---

## Prefer JSDoc over inline comments

- use JSDoc (`/** */`) for documenting functions, classes, and complex variables
- avoid inline comments (`//`) unless absolutely necessary (e.g. explaining non-obvious logic)
- JSDoc provides better IDE support, type hints, and is more structured
- keep JSDoc short — one line for simple things, max 2-3 lines for complex logic

### ❌ Bad (verbose JSDoc)

```ts
/**
 * This function is responsible for creating a new user entity
 * in the system. It takes the user's name as a parameter and
 * returns a newly constructed user object containing all
 * the provided information.
 *
 * @param name - The full name of the user that will be stored
 * @returns A new user object with the name property set
 */
function createUser(name: string) {
  return { name }
}
```

### ❌ Bad
```ts
// creates a new user
function createUser(name: string) {
  return { name }
}
````

### ❌ Bad

```ts
const timeout = 5000 // timeout in ms
```

### ✅ Good

```ts
/**
 * Creates a new user
 * @param name - User name
 */
function createUser(name: string) {
  return { name }
}
```

### ✅ Good

```ts
/**
 * Timeout in milliseconds for external API calls
 */
const TIMEOUT_MS = 5000
```

### ✅ Good (complex logic explanation)

```ts
/**
 * Calculates final price with discount and tax
 */
function calculatePrice(price: number) {
  // Apply discount before tax due to business requirement
  const discounted = price * 0.9
  return discounted * 1.21
}
```

## 🗺️ Maps

### Keep `Map` JSDoc minimal (one-liner)

* Every `Map` should have a **single-line JSDoc** describing key -> value
* Especially required for `Map<string, string>`

#### ❌ Bad

```ts
const userLabels = new Map<string, string>()
```

#### ✅ Good

```ts
/** `externalProviderId` -> `internalUserId` */
const providerToUserId = new Map<string, string>()
```

### Return awaited promises from `async` functions

* In `async` functions, use `return await ...` instead of returning a raw promise
* This keeps error handling and stack traces consistent

#### ❌ Bad

```ts
async getUser(id: string) {
  return this.userService.findById(id)
}
```

#### ✅ Good

```ts
async getUser(id: string) {
  return await this.userService.findById(id)
}
```

## 🕸️ GraphQL (GQL)

### Do not expose related entity IDs

* Never expose foreign keys like `userId`, `patientId`, `reservationId` in GraphQL DTOs/types
* Expose related entities as nested fields instead

#### ❌ Bad

```ts
@ObjectType("Reservation")
class ReservationModel {
  @Field(() => String)
  patientId: string
}
```

#### ✅ Good

```ts
@ResolveField(() => PatientModel)
async patient(
  @Parent() reservation: ReservationModel,
  @Context() { loaders }: LoadersContext,
) {
  return await loaders.reservation.patientLoader.load(reservation.patientId)
}
```

### Resolve relations via `@ResolveField` + DataLoader

* Related entities must be resolved through `@ResolveField`
* Async relation resolvers must always use DataLoader (no direct service/repository calls inside resolver)

#### ❌ Bad

```ts
@ResolveField(() => PatientGql)
async patient(@Parent() reservation: ReservationModel) {
  return await this.patientService.getById(reservation.patientId)
}
```

#### ✅ Good

```ts
@ResolveField(() => PatientModel)
async patient(
  @Parent() reservation: ReservationModel,
  @Context() { loaders }: LoadersContext,
) {
  return await loaders.reservation.patientLoader.load(reservation.patientId)
}
```

---

### Keep GraphQL shape aligned with DB relation structure

* GraphQL should mirror real entity relationships
* Do not hide connection/join entities (tables) when they carry relational meaning
* If relation is modeled through a link entity/table, represent that explicitly in GraphQL schema

#### ❌ Bad

```ts
// hides Membership link table
User.projects: [Project]
```

#### ✅ Good

```ts
User.memberships: [Membership]
Membership.project: Project
Membership.user: User
```

## 🔁 Loops & Mandatory Entities

### Do not silently `continue` when a mandatory entity is missing

* If an entity is required for processing, never skip it silently inside a loop
* Either:
  * throw an error, or
  * atleast log with entity IDs

#### ❌ Bad

```ts
for (const reservationId of reservationIds) {
  const reservation = reservationById.get(reservationId)
  if (!reservation) {
    continue
  }

  processed.push(reservation)
}
```

#### ✅ Good

```ts
for (const reservationId of reservationIds) {
  const reservation = reservationById.get(reservationId)
  if (!reservation) {
    this.logger.error('Mandatory reservation missing', { reservationId })
    throw new Error(`Mandatory reservation missing: ${reservationId}`)
  }

  processed.push(reservation)
}
```

### Do not use `logger.debug` for important events

* `logger.debug` is often disabled in production and can hide critical runtime information
* Use `logger.log`, `logger.warn`, or `logger.error` based on severity

#### ❌ Bad

```ts
this.logger.debug('Payment sync failed', { paymentId })
```

#### ✅ Good

```ts
this.logger.error('Payment sync failed', { paymentId })
```

#### ✅ Good

```ts
this.logger.log('Payment sync started', { paymentId })
```
