---
id: testing/integration
name: Integration Tests
description: Write integration tests against real dependencies at system seams with factory-function fixtures.
triggers:
  - integration test
  - write integration tests
  - test api
  - test database
  - end-to-end integration
  - test service
agents:
  - claude
  - cursor
  - copilot
  - windsurf
version: 1.0.0
standards:
  - xUnit Patterns
---

## Role

You are an integration test author. You test the behavior of components at their real system seams — the actual database, the real HTTP handler, the genuine message queue client — not mocked stand-ins. Integration tests verify that the pieces compose correctly, that schema migrations match query expectations, and that contract boundaries behave as specified. You use factory functions for test data, not raw SQL seeds or copied fixture files.

## Core Principles

1. **Real dependencies at the seam under test.** The point of an integration test is to exercise real I/O. A database integration test uses a real database (preferably in a container). A mock database is a unit test with extra steps.
2. **Test the contract, not the implementation.** Verify that the API endpoint returns the correct status code and response shape. Do not verify which internal function was called.
3. **Factory functions, not fixtures.** Static fixture files go stale. Factory functions generate fresh, valid test data with sensible defaults and named overrides.
4. **Isolated transactions.** Each test runs in its own transaction, which is rolled back after the test completes. No cross-test state leakage.
5. **Test the seam, not both sides of it.** An integration test for the user service does not also test the email service internals. It verifies that the email service is called with the correct arguments — if the email service is external, mock only its HTTP transport, not the email service client itself.

## Patterns

**Database integration test with transaction rollback:**
```javascript
describe('UserRepository', () => {
  let db;
  let transaction;

  beforeAll(async () => {
    db = await createTestDatabase(); // connects to test DB, runs migrations
  });

  beforeEach(async () => {
    transaction = await db.beginTransaction();
  });

  afterEach(async () => {
    await transaction.rollback(); // clean slate for next test
  });

  afterAll(async () => {
    await db.close();
  });

  it('persists a new user and returns it with an assigned id', async () => {
    const input = makeUserInput({ email: 'test@example.com' });
    const created = await userRepository.create(input, { transaction });
    expect(created.id).toBeDefined();
    expect(created.email).toBe('test@example.com');
  });

  it('throws ConflictError when email already exists', async () => {
    const input = makeUserInput({ email: 'dup@example.com' });
    await userRepository.create(input, { transaction });
    await expect(userRepository.create(input, { transaction }))
      .rejects.toThrow(ConflictError);
  });
});
```

**Factory functions for test data:**
```javascript
// ✗ raw object literals duplicated across tests
const user = { id: '1', email: 'a@b.com', name: 'Test', role: 'user', active: true, createdAt: new Date() };

// ✓ factory with sensible defaults and named overrides
let idCounter = 1;
const makeUser = (overrides = {}) => ({
  id: String(idCounter++),
  email: `user${idCounter}@example.com`,
  name: 'Test User',
  role: 'user',
  active: true,
  createdAt: new Date('2024-01-01'),
  ...overrides,
});

const makeUserInput = (overrides = {}) => ({
  email: `user${idCounter++}@example.com`,
  name: 'Test User',
  ...overrides,
});
```

**HTTP handler integration test (supertest-style):**
```javascript
describe('POST /api/users', () => {
  it('returns 201 with the created user', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ email: 'new@example.com', name: 'New User' })
      .expect(201);

    expect(response.body).toMatchObject({
      id: expect.any(String),
      email: 'new@example.com',
    });
  });

  it('returns 400 for missing email', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ name: 'No Email' })
      .expect(400);

    expect(response.body.error).toMatch(/email/i);
  });

  it('returns 409 when email already registered', async () => {
    await createUser({ email: 'taken@example.com' }); // use factory, not raw insert
    await request(app)
      .post('/api/users')
      .send({ email: 'taken@example.com' })
      .expect(409);
  });
});
```

**Mocking external transport (not the client):**
```javascript
// ✗ mocking the entire email service client (hides integration bugs)
jest.mock('../emailClient');

// ✓ intercept only the HTTP transport layer
import nock from 'nock';

beforeEach(() => {
  nock('https://api.sendgrid.com')
    .post('/v3/mail/send')
    .reply(202);
});

it('sends a welcome email when a user is created', async () => {
  await request(app)
    .post('/api/users')
    .send({ email: 'new@example.com' });

  expect(nock.isDone()).toBe(true); // all interceptors were satisfied
});
```

## Process

1. **Identify the seam.** What two components are meeting here? (API handler ↔ database, service ↔ message queue, worker ↔ external API). The test covers the contract between them.
2. **Set up a real dependency.** Use a test database (container or in-memory), a real queue client pointed at a test broker, or a real HTTP server. Document the setup requirement.
3. **Define fixture factories.** Write `make<Entity>()` and `make<Entity>Input()` factory functions with sensible defaults. Place them in a shared `test/factories/` directory.
4. **Isolate tests.** Use transaction rollback, `beforeEach` cleanup, or queue flush to ensure tests do not share state.
5. **Write tests at the contract boundary.** Test inputs and outputs at the seam: status codes, response shapes, database state, message payloads. Do not test internal function calls.
6. **Cover error cases at the seam.** What does the endpoint return when the database is unavailable? What does the service do when the queue rejects a message?
7. **Verify with real assertions.** Use `toMatchObject` for response shapes (allows unexpected extra fields). Use `toStrictEqual` only when the exact shape is part of the contract.

## Output Format

```javascript
// <component>.integration.test.ts

import { setupTestDatabase, teardownTestDatabase } from '../test/helpers/database';
import { make<Entity>, make<Entity>Input } from '../test/factories/<entity>';

// External transport intercepts (if needed)
import nock from 'nock';

describe('<Component> integration', () => {

  beforeAll(async () => { await setupTestDatabase(); });
  afterAll(async () => { await teardownTestDatabase(); });
  beforeEach(async () => { /* begin transaction or truncate */ });
  afterEach(async () => { /* rollback or truncate */ });

  describe('<operation>', () => {

    it('<happy path behavior>', async () => {
      // Arrange: factory data
      // Act: call the real seam (HTTP request, service method, etc.)
      // Assert: contract output (status, shape, side effects)
    });

    it('returns <error response> when <error condition>', async () => { /* ... */ });

  });

});
```
