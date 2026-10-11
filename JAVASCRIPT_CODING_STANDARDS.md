# JavaScript Coding Standards & Best Practices

> A practical guide for writing clean, maintainable, and production-ready JavaScript/Node.js code.
> Every rule includes a ❌ Bad and ✅ Good example.

---

## Table of Contents

- [1. Variables & Constants](#1-variables--constants)
- [2. Functions](#2-functions)
- [3. Naming Conventions](#3-naming-conventions)
- [4. Conditionals & Control Flow](#4-conditionals--control-flow)
- [5. Objects & Arrays](#5-objects--arrays)
- [6. Destructuring](#6-destructuring)
- [7. Template Literals](#7-template-literals)
- [8. Arrow Functions](#8-arrow-functions)
- [9. Async/Await & Promises](#9-asyncawait--promises)
- [10. Error Handling](#10-error-handling)
- [11. Modules (Import/Export)](#11-modules-importexport)
- [12. Classes & OOP](#12-classes--oop)
- [13. Comments & Documentation](#13-comments--documentation)
- [14. API Design (Express.js)](#14-api-design-expressjs)
- [15. Database (PostgreSQL/SQL)](#15-database-postgresqlsql)
- [16. Security](#16-security)
- [17. Performance](#17-performance)
- [18. Testing (Jest)](#18-testing-jest)
- [19. Project Structure](#19-project-structure)
- [20. Code Formatting & Linting](#20-code-formatting--linting)

---

## 1. Variables & Constants

### 1.1 Use `const` by default, `let` when reassignment is needed. Never use `var`.

```js
// ❌ Bad
var userName = 'Belshick';
var age = 28;
var isActive = true;

// ✅ Good
const userName = 'Belshick';
let age = 28;          // age may change, so let is fine
const isActive = true;
```

**Why:** `var` is function-scoped and gets hoisted, causing unexpected bugs. `const` and `let` are block-scoped and predictable.

### 1.2 Declare one variable per line

```js
// ❌ Bad
const firstName = 'Belshick', lastName = 'Megason', role = 'developer';

// ✅ Good
const firstName = 'Belshick';
const lastName = 'Megason';
const role = 'developer';
```

### 1.3 Group `const` declarations first, then `let`

```js
// ❌ Bad
let count = 0;
const MAX_RETRIES = 3;
let isLoading = false;
const BASE_URL = '/api/v1';

// ✅ Good
const MAX_RETRIES = 3;
const BASE_URL = '/api/v1';
let count = 0;
let isLoading = false;
```

### 1.4 Avoid magic numbers — use named constants

```js
// ❌ Bad
if (password.length < 8) {
  throw new Error('Password too short');
}

setTimeout(fetchData, 300000);

// ✅ Good
const MIN_PASSWORD_LENGTH = 8;
const FIVE_MINUTES_MS = 5 * 60 * 1000;

if (password.length < MIN_PASSWORD_LENGTH) {
  throw new Error('Password too short');
}

setTimeout(fetchData, FIVE_MINUTES_MS);
```

---

## 2. Functions

### 2.1 Keep functions small — one function, one job

```js
// ❌ Bad — doing too many things
function processUser(userData) {
  // validate
  if (!userData.email) throw new Error('Email required');
  if (!userData.name) throw new Error('Name required');

  // format
  userData.email = userData.email.toLowerCase().trim();
  userData.name = userData.name.trim();

  // save to database
  db.query('INSERT INTO users (name, email) VALUES ($1, $2)', [userData.name, userData.email]);

  // send welcome email
  emailService.send(userData.email, 'Welcome!', 'Thanks for joining');

  // log
  logger.info(`User created: ${userData.email}`);
}

// ✅ Good — each function does one thing
function validateUser(userData) {
  if (!userData.email) throw new Error('Email required');
  if (!userData.name) throw new Error('Name required');
}

function formatUser(userData) {
  return {
    email: userData.email.toLowerCase().trim(),
    name: userData.name.trim(),
  };
}

async function saveUser(userData) {
  return db.query('INSERT INTO users (name, email) VALUES ($1, $2)', [userData.name, userData.email]);
}

async function processUser(userData) {
  validateUser(userData);
  const formatted = formatUser(userData);
  await saveUser(formatted);
  await emailService.send(formatted.email, 'Welcome!', 'Thanks for joining');
  logger.info(`User created: ${formatted.email}`);
}
```

### 2.2 Use default parameters instead of manual checks

```js
// ❌ Bad
function createUser(name, role) {
  const userRole = role || 'viewer';
  const userName = name || 'Anonymous';
  // ...
}

// ✅ Good
function createUser(name = 'Anonymous', role = 'viewer') {
  // ...
}
```

### 2.3 Limit function parameters — use an object for 3+

```js
// ❌ Bad — too many parameters, order-dependent
function createAppointment(patientUid, doctorUid, date, time, duration, notes, status) {
  // ...
}
createAppointment(101, 55, '2026-01-15', '10:00', 30, 'Follow-up', 'scheduled');

// ✅ Good — single config object
function createAppointment({ patientUid, doctorUid, date, time, duration = 30, notes = '', status = 'scheduled' }) {
  // ...
}
createAppointment({
  patientUid: 101,
  doctorUid: 55,
  date: '2026-01-15',
  time: '10:00',
  notes: 'Follow-up',
});
```

### 2.4 Return early to avoid deep nesting

```js
// ❌ Bad — pyramid of doom
function getDiscount(user) {
  if (user) {
    if (user.isActive) {
      if (user.isPremium) {
        return 0.2;
      } else {
        return 0.1;
      }
    } else {
      return 0;
    }
  } else {
    return 0;
  }
}

// ✅ Good — early returns
function getDiscount(user) {
  if (!user) return 0;
  if (!user.isActive) return 0;
  if (user.isPremium) return 0.2;
  return 0.1;
}
```

### 2.5 Avoid side effects — functions should be predictable

```js
// ❌ Bad — mutates external state
let totalPrice = 0;

function addItem(price) {
  totalPrice += price;  // modifies external variable
}

// ✅ Good — pure function
function calculateTotal(items) {
  return items.reduce((total, item) => total + item.price, 0);
}
```

---

## 3. Naming Conventions

### 3.1 Use meaningful, descriptive names

```js
// ❌ Bad
const d = new Date();
const u = getUser();
const val = calculate(x, y);
const arr = getData();
const flag = true;

// ✅ Good
const currentDate = new Date();
const currentUser = getUser();
const totalRevenue = calculate(unitPrice, quantity);
const activePatients = getData();
const isEligibleForDiscount = true;
```

### 3.2 Follow consistent naming conventions

```js
// ❌ Bad — inconsistent casing
const user_name = 'Belshick';       // snake_case in JS
const GetUserData = () => {};        // PascalCase for function
const IS_ACTIVE = true;             // UPPER_CASE for non-constant variable

// ✅ Good
const userName = 'Belshick';                  // camelCase for variables
const getUserData = () => {};                 // camelCase for functions
const MAX_RETRY_COUNT = 3;                    // UPPER_SNAKE_CASE for true constants
class UserService {}                          // PascalCase for classes
const isActive = true;                        // prefix booleans with is/has/can/should
```

### 3.3 Boolean names should read like a question

```js
// ❌ Bad
const active = true;
const login = false;
const permission = true;
const data = false;

// ✅ Good
const isActive = true;
const isLoggedIn = false;
const hasPermission = true;
const hasData = false;
```

### 3.4 Function names should describe the action

```js
// ❌ Bad
function data() {}
function process() {}
function handle() {}
function user(id) {}

// ✅ Good
function fetchRadiologistData() {}
function processAppointment() {}
function handleLoginRequest() {}
function getUserById(id) {}
```

---

## 4. Conditionals & Control Flow

### 4.1 Use strict equality (`===`) always

```js
// ❌ Bad — loose equality causes type coercion bugs
if (status == '1') {}
if (value == null) {}
if (count == 0) {}

// ✅ Good
if (status === '1') {}
if (value === null || value === undefined) {}
if (count === 0) {}
```

### 4.2 Use ternary for simple assignments, not complex logic

```js
// ❌ Bad — nested ternary is unreadable
const label = role === 'admin' ? 'Administrator' : role === 'editor' ? 'Editor' : role === 'viewer' ? 'Viewer' : 'Unknown';

// ✅ Good — use a map or switch for multiple conditions
const ROLE_LABELS = {
  admin: 'Administrator',
  editor: 'Editor',
  viewer: 'Viewer',
};
const label = ROLE_LABELS[role] || 'Unknown';

// ✅ Simple ternary is fine
const statusText = isActive ? 'Active' : 'Inactive';
```

### 4.3 Use optional chaining (`?.`) and nullish coalescing (`??`)

```js
// ❌ Bad
const city = user && user.address && user.address.city ? user.address.city : 'Unknown';
const port = config.port || 3000;  // fails if port is 0 (falsy but valid)

// ✅ Good
const city = user?.address?.city ?? 'Unknown';
const port = config.port ?? 3000;  // only defaults if null/undefined, not 0
```

### 4.4 Avoid negative conditionals

```js
// ❌ Bad
if (!isNotActive) {}
if (!disableValidation) {}

// ✅ Good
if (isActive) {}
if (enableValidation) {}
```

---

## 5. Objects & Arrays

### 5.1 Use object shorthand

```js
// ❌ Bad
const name = 'Belshick';
const role = 'developer';
const user = {
  name: name,
  role: role,
  getFullName: function() {
    return this.name;
  }
};

// ✅ Good
const name = 'Belshick';
const role = 'developer';
const user = {
  name,
  role,
  getFullName() {
    return this.name;
  }
};
```

### 5.2 Use spread operator instead of mutation

```js
// ❌ Bad — mutates the original object
const user = { name: 'Belshick', role: 'developer' };
user.role = 'senior developer';
user.department = 'engineering';

// ✅ Good — creates a new object
const user = { name: 'Belshick', role: 'developer' };
const updatedUser = {
  ...user,
  role: 'senior developer',
  department: 'engineering',
};
```

### 5.3 Use array methods over loops

```js
// ❌ Bad
const activeUsers = [];
for (let i = 0; i < users.length; i++) {
  if (users[i].isActive) {
    activeUsers.push(users[i].name.toUpperCase());
  }
}

// ✅ Good
const activeUsers = users
  .filter(user => user.isActive)
  .map(user => user.name.toUpperCase());
```

### 5.4 Use `Array.includes()` instead of multiple OR checks

```js
// ❌ Bad
if (status === 'pending' || status === 'processing' || status === 'queued') {
  // ...
}

// ✅ Good
const PENDING_STATUSES = ['pending', 'processing', 'queued'];
if (PENDING_STATUSES.includes(status)) {
  // ...
}
```

### 5.5 Use `Object.freeze()` for truly immutable config

```js
// ❌ Bad — can be accidentally modified
const config = {
  apiVersion: 'v1',
  maxRetries: 3,
  timeout: 5000,
};
config.maxRetries = 100; // no error, silently mutated

// ✅ Good
const config = Object.freeze({
  apiVersion: 'v1',
  maxRetries: 3,
  timeout: 5000,
});
config.maxRetries = 100; // throws error in strict mode
```

---

## 6. Destructuring

### 6.1 Destructure objects and arrays

```js
// ❌ Bad
const firstName = user.firstName;
const lastName = user.lastName;
const email = user.email;
const firstItem = items[0];
const secondItem = items[1];

// ✅ Good
const { firstName, lastName, email } = user;
const [firstItem, secondItem] = items;
```

### 6.2 Destructure function parameters

```js
// ❌ Bad
function getFullName(user) {
  return `${user.firstName} ${user.lastName}`;
}

// ✅ Good
function getFullName({ firstName, lastName }) {
  return `${firstName} ${lastName}`;
}
```

### 6.3 Use default values in destructuring

```js
// ❌ Bad
function createConfig(options) {
  const port = options.port || 3000;
  const host = options.host || 'localhost';
  const debug = options.debug || false;
}

// ✅ Good
function createConfig({ port = 3000, host = 'localhost', debug = false } = {}) {
  // port, host, debug are ready to use with defaults
}
```

---

## 7. Template Literals

### 7.1 Use template literals for string concatenation

```js
// ❌ Bad
const message = 'Hello ' + firstName + ' ' + lastName + ', welcome to ' + appName + '!';
const url = baseUrl + '/api/' + version + '/users/' + userId;

// ✅ Good
const message = `Hello ${firstName} ${lastName}, welcome to ${appName}!`;
const url = `${baseUrl}/api/${version}/users/${userId}`;
```

### 7.2 Use tagged template literals for SQL (preventing injection)

```js
// ❌ Bad — SQL injection vulnerability
const query = `SELECT * FROM users WHERE email = '${email}'`;

// ✅ Good — parameterized query
const query = {
  text: 'SELECT * FROM users WHERE email = $1',
  values: [email],
};
```

---

## 8. Arrow Functions

### 8.1 Use arrow functions for callbacks and anonymous functions

```js
// ❌ Bad
const doubled = numbers.map(function(num) {
  return num * 2;
});

setTimeout(function() {
  console.log('Done');
}, 1000);

// ✅ Good
const doubled = numbers.map(num => num * 2);

setTimeout(() => console.log('Done'), 1000);
```

### 8.2 Use implicit return for single expressions

```js
// ❌ Bad
const getFullName = (user) => {
  return `${user.firstName} ${user.lastName}`;
};

const isAdult = (age) => {
  return age >= 18;
};

// ✅ Good
const getFullName = (user) => `${user.firstName} ${user.lastName}`;
const isAdult = (age) => age >= 18;
```

### 8.3 Don't use arrow functions as object methods

```js
// ❌ Bad — `this` doesn't bind to the object in arrow functions
const user = {
  name: 'Belshick',
  greet: () => {
    console.log(`Hi, I'm ${this.name}`); // `this` is undefined
  },
};

// ✅ Good
const user = {
  name: 'Belshick',
  greet() {
    console.log(`Hi, I'm ${this.name}`); // `this` correctly refers to user
  },
};
```

---

## 9. Async/Await & Promises

### 9.1 Use async/await over `.then()` chains

```js
// ❌ Bad — callback/promise chain hell
function getPatientReport(patientId) {
  return getPatient(patientId)
    .then(patient => {
      return getAppointments(patient.uid)
        .then(appointments => {
          return getReports(appointments[0].uid)
            .then(reports => {
              return { patient, appointments, reports };
            });
        });
    })
    .catch(err => console.error(err));
}

// ✅ Good — flat and readable
async function getPatientReport(patientId) {
  try {
    const patient = await getPatient(patientId);
    const appointments = await getAppointments(patient.uid);
    const reports = await getReports(appointments[0].uid);
    return { patient, appointments, reports };
  } catch (error) {
    logger.error('Failed to get patient report', { patientId, error: error.message });
    throw error;
  }
}
```

### 9.2 Use `Promise.all()` for independent async operations

```js
// ❌ Bad — sequential when they could be parallel
async function getDashboardData(userId) {
  const profile = await getProfile(userId);       // waits 200ms
  const notifications = await getNotifications(userId);  // waits 150ms
  const appointments = await getAppointments(userId);    // waits 300ms
  return { profile, notifications, appointments };
  // Total: ~650ms
}

// ✅ Good — parallel execution
async function getDashboardData(userId) {
  const [profile, notifications, appointments] = await Promise.all([
    getProfile(userId),
    getNotifications(userId),
    getAppointments(userId),
  ]);
  return { profile, notifications, appointments };
  // Total: ~300ms (slowest one)
}
```

### 9.3 Use `Promise.allSettled()` when some failures are acceptable

```js
// ❌ Bad — one failure cancels everything
const [users, logs, metrics] = await Promise.all([
  fetchUsers(),    // if this fails, logs and metrics also fail
  fetchLogs(),
  fetchMetrics(),
]);

// ✅ Good — each result handled independently
const results = await Promise.allSettled([
  fetchUsers(),
  fetchLogs(),
  fetchMetrics(),
]);

const users = results[0].status === 'fulfilled' ? results[0].value : [];
const logs = results[1].status === 'fulfilled' ? results[1].value : [];
const metrics = results[2].status === 'fulfilled' ? results[2].value : [];
```

### 9.4 Never use `async` without `await`

```js
// ❌ Bad — async is pointless here
async function getFullName(user) {
  return `${user.firstName} ${user.lastName}`;
}

// ✅ Good — no async needed for synchronous code
function getFullName(user) {
  return `${user.firstName} ${user.lastName}`;
}
```

---

## 10. Error Handling

### 10.1 Always use try/catch with async/await

```js
// ❌ Bad — unhandled promise rejection
async function getUser(id) {
  const user = await db.query('SELECT * FROM users WHERE uid = $1', [id]);
  return user.rows[0];
}

// ✅ Good
async function getUser(id) {
  try {
    const result = await db.query('SELECT * FROM users WHERE uid = $1', [id]);
    if (!result.rows.length) {
      throw new AppError('User not found', 404);
    }
    return result.rows[0];
  } catch (error) {
    logger.error('getUser failed', { id, error: error.message });
    throw error;
  }
}
```

### 10.2 Create custom error classes

```js
// ❌ Bad — generic errors with no structure
throw new Error('Not found');
throw new Error('Unauthorized');
throw new Error('Validation failed');

// ✅ Good — custom error class
class AppError extends Error {
  constructor(message, statusCode, errorCode = null) {
    super(message);
    this.name = 'AppError';
    this.statusCode = statusCode;
    this.errorCode = errorCode;
    this.isOperational = true;
    Error.captureStackTrace(this, this.constructor);
  }
}

class NotFoundError extends AppError {
  constructor(resource = 'Resource') {
    super(`${resource} not found`, 404, 'NOT_FOUND');
  }
}

class ValidationError extends AppError {
  constructor(message) {
    super(message, 400, 'VALIDATION_ERROR');
  }
}

// Usage
throw new NotFoundError('Patient');
throw new ValidationError('Email is required');
```

### 10.3 Centralized error handling middleware (Express)

```js
// ❌ Bad — error handling in every route
app.get('/users/:id', async (req, res) => {
  try {
    const user = await getUser(req.params.id);
    res.json(user);
  } catch (error) {
    if (error.statusCode === 404) {
      res.status(404).json({ error: error.message });
    } else if (error.statusCode === 400) {
      res.status(400).json({ error: error.message });
    } else {
      res.status(500).json({ error: 'Internal server error' });
    }
  }
});

// ✅ Good — centralized error handler
// Async wrapper
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// Route is clean
app.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await getUser(req.params.id);
  res.json(user);
}));

// One error handler for the entire app
app.use((error, req, res, next) => {
  const statusCode = error.statusCode || 500;
  const message = error.isOperational ? error.message : 'Internal server error';

  logger.error('Request error', {
    method: req.method,
    url: req.originalUrl,
    statusCode,
    error: error.message,
    stack: error.stack,
  });

  res.status(statusCode).json({
    success: false,
    error: message,
    ...(process.env.NODE_ENV === 'development' && { stack: error.stack }),
  });
});
```

### 10.4 Never swallow errors silently

```js
// ❌ Bad — error is caught and ignored
try {
  await saveUser(userData);
} catch (error) {
  // silently ignored — nobody will ever know this failed
}

// ❌ Bad — only logs, doesn't handle
try {
  await saveUser(userData);
} catch (error) {
  console.log(error);
}

// ✅ Good — log with context and re-throw or handle
try {
  await saveUser(userData);
} catch (error) {
  logger.error('Failed to save user', {
    userId: userData.uid,
    error: error.message,
  });
  throw error;
}
```

---

## 11. Modules (Import/Export)

### 11.1 Use named exports for utilities, default export for main class/function

```js
// ❌ Bad — everything as default
// utils.js
export default {
  formatDate(date) { /* ... */ },
  formatCurrency(amount) { /* ... */ },
  formatName(name) { /* ... */ },
};

// ✅ Good — named exports (tree-shakeable)
// utils.js
export function formatDate(date) { /* ... */ }
export function formatCurrency(amount) { /* ... */ }
export function formatName(name) { /* ... */ }

// Import only what you need
import { formatDate, formatName } from './utils.js';
```

### 11.2 One module, one responsibility

```js
// ❌ Bad — everything dumped in one file
// helpers.js (500 lines with date, string, auth, db, email helpers)

// ✅ Good — organized by concern
// utils/date.js
// utils/string.js
// utils/validation.js
// services/auth.service.js
// services/email.service.js
```

---

## 12. Classes & OOP

### 12.1 Use classes with proper encapsulation

```js
// ❌ Bad — no encapsulation
class UserService {
  constructor() {
    this.db = getDBConnection();
  }
}
const service = new UserService();
service.db.query('DROP TABLE users'); // direct DB access from outside

// ✅ Good — private fields and controlled access
class UserService {
  #db;

  constructor(db) {
    this.#db = db;  // private — cannot be accessed from outside
  }

  async getUserById(id) {
    const result = await this.#db.query('SELECT * FROM users WHERE uid = $1', [id]);
    return result.rows[0] ?? null;
  }
}
```

### 12.2 Prefer composition over deep inheritance

```js
// ❌ Bad — deep inheritance chain
class Animal {}
class Mammal extends Animal {}
class DomesticMammal extends Mammal {}
class Dog extends DomesticMammal {}
class GuideDog extends Dog {}

// ✅ Good — compose behaviors
class Animal {
  constructor(name) {
    this.name = name;
  }
}

const canWalk = (animal) => ({
  walk: () => `${animal.name} is walking`,
});

const canGuide = (animal) => ({
  guide: (person) => `${animal.name} is guiding ${person}`,
});

function createGuideDog(name) {
  const dog = new Animal(name);
  return Object.assign(dog, canWalk(dog), canGuide(dog));
}
```

---

## 13. Comments & Documentation

### 13.1 Write self-documenting code — comments explain WHY, not WHAT

```js
// ❌ Bad — comment states the obvious
// Get user by id
function getUserById(id) {
  // Check if id exists
  if (!id) {
    // Throw error
    throw new Error('ID required');
  }
  // Return user from database
  return db.query('SELECT * FROM users WHERE uid = $1', [id]);
}

// ✅ Good — comment explains business logic / WHY
function getUserById(id) {
  if (!id) {
    throw new ValidationError('ID required');
  }

  // Using uid instead of email lookup because uid is indexed
  // and email has a unique constraint that causes slower scans
  return db.query('SELECT * FROM users WHERE uid = $1', [id]);
}
```

### 13.2 Use JSDoc for functions that are part of a public API

```js
// ❌ Bad — no documentation on a shared utility
function calculateAge(dob) {
  const diff = Date.now() - new Date(dob).getTime();
  return Math.floor(diff / 31557600000);
}

// ✅ Good
/**
 * Calculates age in full years from a date of birth.
 * @param {string|Date} dob - Date of birth (ISO string or Date object)
 * @returns {number} Age in completed years
 * @example
 * calculateAge('1998-05-15'); // 28
 */
function calculateAge(dob) {
  const diff = Date.now() - new Date(dob).getTime();
  return Math.floor(diff / (365.25 * 24 * 60 * 60 * 1000));
}
```

### 13.3 Use TODO/FIXME with owner and ticket

```js
// ❌ Bad
// TODO: fix this later
// FIXME: this is broken

// ✅ Good
// TODO(belshick): Add pagination — JIRA-1234
// FIXME(belshick): Race condition when concurrent updates — JIRA-5678
```

---

## 14. API Design (Express.js)

### 14.1 Use consistent response format

```js
// ❌ Bad — inconsistent response shapes
app.get('/users', (req, res) => res.json(users));
app.get('/users/:id', (req, res) => res.json({ user }));
app.post('/users', (req, res) => res.json({ message: 'Created', id: user.id }));
app.get('/error', (req, res) => res.status(400).json('Bad request'));

// ✅ Good — consistent response envelope
const sendSuccess = (res, data, statusCode = 200) => {
  res.status(statusCode).json({
    success: true,
    data,
  });
};

const sendError = (res, message, statusCode = 400) => {
  res.status(statusCode).json({
    success: false,
    error: message,
  });
};

app.get('/users', asyncHandler(async (req, res) => {
  const users = await userService.getAll();
  sendSuccess(res, users);
}));

app.post('/users', asyncHandler(async (req, res) => {
  const user = await userService.create(req.body);
  sendSuccess(res, user, 201);
}));
```

### 14.2 Separate route, controller, service, and repository layers

```js
// ❌ Bad — everything in the route file
app.get('/api/v1/radiologists/:id', async (req, res) => {
  try {
    const result = await db.query('SELECT * FROM settings.radiologists WHERE uid = $1', [req.params.id]);
    if (!result.rows.length) return res.status(404).json({ error: 'Not found' });
    res.json(result.rows[0]);
  } catch (err) {
    res.status(500).json({ error: 'Server error' });
  }
});

// ✅ Good — layered architecture

// routes/radiologist.routes.js
router.get('/:id', asyncHandler(radiologistController.getById));

// controllers/radiologist.controller.js
const getById = async (req, res) => {
  const radiologist = await radiologistService.getById(req.params.id);
  sendSuccess(res, radiologist);
};

// services/radiologist.service.js
const getById = async (id) => {
  const radiologist = await radiologistRepository.findById(id);
  if (!radiologist) throw new NotFoundError('Radiologist');
  return radiologist;
};

// repositories/radiologist.repository.js
const findById = async (id) => {
  const result = await db.query('SELECT * FROM settings.radiologists WHERE uid = $1', [id]);
  return result.rows[0] ?? null;
};
```

### 14.3 Validate request input at the boundary

```js
// ❌ Bad — no validation
app.post('/users', async (req, res) => {
  const user = await createUser(req.body); // trusts everything blindly
  res.json(user);
});

// ✅ Good — validate with Joi/Zod before processing
const createUserSchema = Joi.object({
  firstName: Joi.string().trim().min(1).max(100).required(),
  lastName: Joi.string().trim().min(1).max(100).required(),
  email: Joi.string().email().required(),
  role: Joi.string().valid('admin', 'viewer', 'editor').default('viewer'),
});

const validate = (schema) => (req, res, next) => {
  const { error, value } = schema.validate(req.body, { abortEarly: false });
  if (error) {
    const messages = error.details.map(d => d.message);
    throw new ValidationError(messages.join(', '));
  }
  req.body = value;  // use sanitized values
  next();
};

app.post('/users', validate(createUserSchema), asyncHandler(async (req, res) => {
  const user = await createUser(req.body);
  sendSuccess(res, user, 201);
}));
```

### 14.4 Use proper HTTP methods and status codes

```js
// ❌ Bad
app.post('/getUser', ...);          // GET action using POST
app.get('/deleteUser/:id', ...);    // DELETE action using GET
res.status(200).json({ error: 'Not found' });  // Error with 200

// ✅ Good
app.get('/users/:id', ...);         // GET for reading
app.post('/users', ...);            // POST for creating → 201
app.put('/users/:id', ...);         // PUT for full update → 200
app.patch('/users/:id', ...);       // PATCH for partial update → 200
app.delete('/users/:id', ...);      // DELETE for removing → 204

// Status codes
// 200 — Success
// 201 — Created
// 204 — No Content (successful delete)
// 400 — Bad Request (validation error)
// 401 — Unauthorized (not logged in)
// 403 — Forbidden (no permission)
// 404 — Not Found
// 409 — Conflict (duplicate)
// 500 — Internal Server Error
```

---

## 15. Database (PostgreSQL/SQL)

### 15.1 Always use parameterized queries

```js
// ❌ Bad — SQL injection vulnerability
const query = `SELECT * FROM users WHERE email = '${email}' AND status = '${status}'`;

// ❌ Bad — string interpolation
const query = `DELETE FROM users WHERE uid = ${userId}`;

// ✅ Good — parameterized
const query = {
  text: 'SELECT * FROM users WHERE email = $1 AND status = $2',
  values: [email, status],
};

// ✅ Good — using pg library
const result = await pool.query(
  'DELETE FROM users WHERE uid = $1 RETURNING *',
  [userId]
);
```

### 15.2 Use connection pooling

```js
// ❌ Bad — new connection per request
async function getUser(id) {
  const client = new Client({ connectionString: DATABASE_URL });
  await client.connect();
  const result = await client.query('SELECT * FROM users WHERE uid = $1', [id]);
  await client.end();
  return result.rows[0];
}

// ✅ Good — connection pool
const { Pool } = require('pg');
const pool = new Pool({
  connectionString: DATABASE_URL,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
});

async function getUser(id) {
  const result = await pool.query('SELECT * FROM users WHERE uid = $1', [id]);
  return result.rows[0];
}
```

### 15.3 Use transactions for multi-step operations

```js
// ❌ Bad — no transaction, partial failure possible
async function transferStudy(studyUid, fromRadiologist, toRadiologist) {
  await db.query('UPDATE studies SET radiologist_uid = $1 WHERE uid = $2', [toRadiologist, studyUid]);
  await db.query('INSERT INTO audit_log (action, study_uid) VALUES ($1, $2)', ['transfer', studyUid]);
  // if second query fails, first already committed — data inconsistency
}

// ✅ Good — transaction ensures all-or-nothing
async function transferStudy(studyUid, fromRadiologist, toRadiologist) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query('UPDATE studies SET radiologist_uid = $1 WHERE uid = $2', [toRadiologist, studyUid]);
    await client.query('INSERT INTO audit_log (action, study_uid) VALUES ($1, $2)', ['transfer', studyUid]);
    await client.query('COMMIT');
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

---

## 16. Security

### 16.1 Never expose sensitive data in responses

```js
// ❌ Bad — sending password hash and internal fields
app.get('/users/:id', async (req, res) => {
  const user = await getUser(req.params.id);
  res.json(user); // includes password_hash, internal_notes, etc.
});

// ✅ Good — whitelist the fields you return
function sanitizeUser(user) {
  const { uid, firstName, lastName, email, role, createdAt } = user;
  return { uid, firstName, lastName, email, role, createdAt };
}

app.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await getUser(req.params.id);
  sendSuccess(res, sanitizeUser(user));
}));
```

### 16.2 Use environment variables for secrets

```js
// ❌ Bad — hardcoded secrets
const JWT_SECRET = 'my-super-secret-key-123';
const DB_PASSWORD = 'admin123';

// ✅ Good — environment variables with validation
const requiredEnvVars = ['JWT_SECRET', 'DATABASE_URL', 'REDIS_URL'];

requiredEnvVars.forEach((key) => {
  if (!process.env[key]) {
    throw new Error(`Missing required environment variable: ${key}`);
  }
});

const config = Object.freeze({
  jwtSecret: process.env.JWT_SECRET,
  databaseUrl: process.env.DATABASE_URL,
  redisUrl: process.env.REDIS_URL,
  port: parseInt(process.env.PORT, 10) || 3000,
});
```

### 16.3 Rate limit your APIs

```js
// ❌ Bad — no rate limiting, vulnerable to brute force
app.post('/auth/login', loginHandler);

// ✅ Good
const rateLimit = require('express-rate-limit');

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5,
  message: { success: false, error: 'Too many login attempts. Try again in 15 minutes.' },
  standardHeaders: true,
});

app.post('/auth/login', loginLimiter, loginHandler);
```

### 16.4 Sanitize user input

```js
// ❌ Bad — trusting user input directly
app.get('/search', (req, res) => {
  const results = db.query(`SELECT * FROM products WHERE name LIKE '%${req.query.q}%'`);
});

// ✅ Good — parameterized + sanitized
const sanitizeInput = (input) => input.replace(/[<>"';]/g, '').trim().slice(0, 200);

app.get('/search', asyncHandler(async (req, res) => {
  const searchTerm = sanitizeInput(req.query.q || '');
  const results = await db.query(
    'SELECT * FROM products WHERE name ILIKE $1 LIMIT 50',
    [`%${searchTerm}%`]
  );
  sendSuccess(res, results.rows);
}));
```

---

## 17. Performance

### 17.1 Use Redis caching for frequently accessed data

```js
// ❌ Bad — hits database every single time
async function getRadiologist(id) {
  return db.query('SELECT * FROM settings.radiologists WHERE uid = $1', [id]);
}

// ✅ Good — cache with TTL
const CACHE_TTL = 300; // 5 minutes

async function getRadiologist(id) {
  const cacheKey = `radiologist:${id}`;

  // Check cache first
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  // Cache miss — query DB
  const result = await db.query('SELECT * FROM settings.radiologists WHERE uid = $1', [id]);
  const radiologist = result.rows[0];

  if (radiologist) {
    await redis.setex(cacheKey, CACHE_TTL, JSON.stringify(radiologist));
  }

  return radiologist;
}

// Invalidate cache on update
async function updateRadiologist(id, data) {
  await db.query('UPDATE settings.radiologists SET ... WHERE uid = $1', [id]);
  await redis.del(`radiologist:${id}`);
}
```

### 17.2 Paginate database queries

```js
// ❌ Bad — returns ALL rows
app.get('/patients', async (req, res) => {
  const result = await db.query('SELECT * FROM patients');
  res.json(result.rows); // could be 100,000+ rows
});

// ✅ Good — cursor-based or offset pagination
app.get('/patients', asyncHandler(async (req, res) => {
  const page = Math.max(1, parseInt(req.query.page, 10) || 1);
  const limit = Math.min(100, parseInt(req.query.limit, 10) || 20);
  const offset = (page - 1) * limit;

  const [dataResult, countResult] = await Promise.all([
    db.query('SELECT * FROM patients ORDER BY created_at DESC LIMIT $1 OFFSET $2', [limit, offset]),
    db.query('SELECT COUNT(*) FROM patients'),
  ]);

  sendSuccess(res, {
    patients: dataResult.rows,
    pagination: {
      page,
      limit,
      total: parseInt(countResult.rows[0].count, 10),
      totalPages: Math.ceil(countResult.rows[0].count / limit),
    },
  });
}));
```

### 17.3 Avoid N+1 queries

```js
// ❌ Bad — N+1 problem (1 query + N queries in a loop)
async function getAppointmentsWithDoctors() {
  const appointments = await db.query('SELECT * FROM appointments');

  for (const apt of appointments.rows) {
    const doctor = await db.query('SELECT * FROM doctors WHERE uid = $1', [apt.doctor_uid]);
    apt.doctor = doctor.rows[0];
  }

  return appointments.rows;
}

// ✅ Good — single JOIN query
async function getAppointmentsWithDoctors() {
  const result = await db.query(`
    SELECT a.*, d.first_name AS doctor_first_name, d.last_name AS doctor_last_name
    FROM appointments a
    JOIN doctors d ON d.uid = a.doctor_uid
    ORDER BY a.appointment_date DESC
  `);
  return result.rows;
}
```

---

## 18. Testing (Jest)

### 18.1 Follow the AAA pattern (Arrange, Act, Assert)

```js
// ❌ Bad — no structure
test('user creation', async () => {
  const user = await createUser({ name: 'Test', email: 'test@test.com' });
  expect(user).toBeDefined();
  expect(user.name).toBe('Test');
  expect(user.email).toBe('test@test.com');
  expect(user.role).toBe('viewer');
  const found = await getUserById(user.uid);
  expect(found.name).toBe('Test');
});

// ✅ Good — clear AAA structure
describe('UserService', () => {
  describe('createUser', () => {
    it('should create a user with default role as viewer', async () => {
      // Arrange
      const input = { name: 'Test User', email: 'test@example.com' };

      // Act
      const user = await userService.createUser(input);

      // Assert
      expect(user).toMatchObject({
        name: 'Test User',
        email: 'test@example.com',
        role: 'viewer',
      });
      expect(user.uid).toBeDefined();
    });

    it('should throw ValidationError when email is missing', async () => {
      // Arrange
      const input = { name: 'Test User' };

      // Act & Assert
      await expect(userService.createUser(input))
        .rejects
        .toThrow(ValidationError);
    });
  });
});
```

### 18.2 Test behavior, not implementation

```js
// ❌ Bad — testing internal implementation
it('should call db.query with correct SQL', async () => {
  await getUser(1);
  expect(db.query).toHaveBeenCalledWith('SELECT * FROM users WHERE uid = $1', [1]);
});

// ✅ Good — testing the behavior/outcome
it('should return user data when user exists', async () => {
  const user = await getUser(1);
  expect(user).toHaveProperty('uid', 1);
  expect(user).toHaveProperty('email');
  expect(user).toHaveProperty('firstName');
});

it('should throw NotFoundError when user does not exist', async () => {
  await expect(getUser(99999)).rejects.toThrow(NotFoundError);
});
```

---

## 19. Project Structure

### 19.1 Organize by feature/module, not by technical role

```
❌ Bad — organized by type (becomes unmanageable at scale)

src/
├── controllers/
│   ├── userController.js
│   ├── appointmentController.js
│   ├── radiologistController.js
│   └── reportController.js
├── models/
│   ├── userModel.js
│   ├── appointmentModel.js
│   └── ...
├── routes/
│   ├── userRoutes.js
│   └── ...
└── services/
    ├── userService.js
    └── ...
```

```
✅ Good — organized by feature/module

src/
├── modules/
│   ├── user/
│   │   ├── user.routes.js
│   │   ├── user.controller.js
│   │   ├── user.service.js
│   │   ├── user.repository.js
│   │   ├── user.validation.js
│   │   └── user.test.js
│   ├── appointment/
│   │   ├── appointment.routes.js
│   │   ├── appointment.controller.js
│   │   ├── appointment.service.js
│   │   ├── appointment.repository.js
│   │   └── appointment.test.js
│   └── radiologist/
│       └── ...
├── common/
│   ├── errors/
│   │   └── AppError.js
│   ├── middleware/
│   │   ├── auth.js
│   │   ├── validate.js
│   │   └── errorHandler.js
│   └── utils/
│       ├── logger.js
│       ├── redis.js
│       └── response.js
├── config/
│   ├── database.js
│   ├── redis.js
│   └── index.js
├── app.js
└── server.js
```

---

## 20. Code Formatting & Linting

### 20.1 Use ESLint + Prettier — non-negotiable

```json
// .eslintrc.json
{
  "env": {
    "node": true,
    "es2022": true,
    "jest": true
  },
  "extends": ["eslint:recommended", "prettier"],
  "parserOptions": {
    "ecmaVersion": "latest",
    "sourceType": "module"
  },
  "rules": {
    "no-console": "warn",
    "no-unused-vars": ["error", { "argsIgnorePattern": "^_" }],
    "prefer-const": "error",
    "no-var": "error",
    "eqeqeq": ["error", "always"],
    "curly": ["error", "all"],
    "no-throw-literal": "error"
  }
}
```

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2
}
```

### 20.2 Pre-commit hooks with Husky + lint-staged

```json
// package.json
{
  "scripts": {
    "lint": "eslint src/",
    "lint:fix": "eslint src/ --fix",
    "format": "prettier --write src/",
    "test": "jest",
    "prepare": "husky install"
  },
  "lint-staged": {
    "*.js": ["eslint --fix", "prettier --write"]
  }
}
```

---

## Quick Reference Checklist

Before every PR, verify:

- [ ] No `var` — only `const` and `let`
- [ ] No magic numbers — named constants used
- [ ] Functions do one thing and are under 30 lines
- [ ] All async functions have try/catch
- [ ] All DB queries are parameterized (no string interpolation)
- [ ] Input validation on every API endpoint
- [ ] No sensitive data in API responses
- [ ] Consistent response format (`{ success, data/error }`)
- [ ] Proper HTTP status codes used
- [ ] Tests follow AAA pattern
- [ ] No `console.log` — use Winston logger
- [ ] Meaningful variable and function names
- [ ] ESLint passes with zero errors

---

> **Author:** Belshick Megason J B
> **Last Updated:** October 2026
> **Stack:** Node.js · TypeScript · Express.js · PostgreSQL · Redis
