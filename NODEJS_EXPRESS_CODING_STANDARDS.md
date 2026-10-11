# Node.js + Express.js Coding Standards & Best Practices

> Production-grade patterns for building scalable, secure, and maintainable backend APIs.
> Every rule includes a ❌ Bad and ✅ Good example.

---

## Table of Contents

- [1. Project Structure](#1-project-structure)
- [2. App Configuration & Startup](#2-app-configuration--startup)
- [3. Environment Variables](#3-environment-variables)
- [4. Routing](#4-routing)
- [5. Controllers](#5-controllers)
- [6. Services (Business Logic)](#6-services-business-logic)
- [7. Repository / Data Access Layer](#7-repository--data-access-layer)
- [8. Middleware](#8-middleware)
- [9. Request Validation](#9-request-validation)
- [10. Error Handling](#10-error-handling)
- [11. Authentication & Authorization](#11-authentication--authorization)
- [12. Database (PostgreSQL)](#12-database-postgresql)
- [13. Redis Caching](#13-redis-caching)
- [14. Logging (Winston)](#14-logging-winston)
- [15. API Response Standards](#15-api-response-standards)
- [16. API Versioning](#16-api-versioning)
- [17. Pagination, Sorting & Filtering](#17-pagination-sorting--filtering)
- [18. File Upload Handling](#18-file-upload-handling)
- [19. Rate Limiting & Throttling](#19-rate-limiting--throttling)
- [20. CORS & Security Headers](#20-cors--security-headers)
- [21. Health Check & Monitoring](#21-health-check--monitoring)
- [22. Graceful Shutdown](#22-graceful-shutdown)
- [23. Process Management (PM2)](#23-process-management-pm2)
- [24. Docker](#24-docker)
- [25. Testing (Jest + Supertest)](#25-testing-jest--supertest)
- [26. API Documentation (Swagger)](#26-api-documentation-swagger)
- [27. Git Commit Standards](#27-git-commit-standards)
- [28. Package.json Scripts](#28-packagejson-scripts)

---

## 1. Project Structure

### 1.1 Organize by feature/module — not by technical role

```
❌ Bad — flat structure grouped by type

src/
├── controllers/
│   ├── userController.js
│   ├── appointmentController.js
│   ├── radiologistController.js
│   └── studyController.js
├── models/
│   ├── userModel.js
│   └── appointmentModel.js
├── routes/
│   ├── userRoutes.js
│   └── appointmentRoutes.js
├── services/
│   └── userService.js
└── app.js
```

```
✅ Good — feature-based modular structure

src/
├── modules/
│   ├── auth/
│   │   ├── auth.routes.js
│   │   ├── auth.controller.js
│   │   ├── auth.service.js
│   │   ├── auth.validation.js
│   │   └── auth.test.js
│   ├── user/
│   │   ├── user.routes.js
│   │   ├── user.controller.js
│   │   ├── user.service.js
│   │   ├── user.repository.js
│   │   ├── user.validation.js
│   │   └── user.test.js
│   ├── radiologist/
│   │   ├── radiologist.routes.js
│   │   ├── radiologist.controller.js
│   │   ├── radiologist.service.js
│   │   ├── radiologist.repository.js
│   │   ├── radiologist.validation.js
│   │   └── radiologist.test.js
│   └── appointment/
│       └── ...
├── common/
│   ├── errors/
│   │   ├── AppError.js
│   │   ├── NotFoundError.js
│   │   ├── ValidationError.js
│   │   └── UnauthorizedError.js
│   ├── middleware/
│   │   ├── auth.middleware.js
│   │   ├── validate.middleware.js
│   │   ├── rateLimiter.middleware.js
│   │   ├── requestLogger.middleware.js
│   │   └── errorHandler.middleware.js
│   ├── utils/
│   │   ├── logger.js
│   │   ├── redis.js
│   │   ├── response.js
│   │   ├── asyncHandler.js
│   │   └── pagination.js
│   └── constants/
│       ├── httpStatus.js
│       └── errorCodes.js
├── config/
│   ├── database.js
│   ├── redis.js
│   ├── cors.js
│   └── index.js
├── database/
│   ├── pool.js
│   ├── migrations/
│   └── seeds/
├── app.js
└── server.js
```

### 1.2 Separate `app.js` and `server.js`

```js
// ❌ Bad — everything in one file
const express = require('express');
const app = express();

app.use(express.json());
app.get('/health', (req, res) => res.json({ status: 'ok' }));

app.listen(3000, () => console.log('Server running on 3000'));

// ✅ Good — app.js exports the app, server.js starts it

// app.js — Express app setup (testable without starting the server)
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const { errorHandler } = require('./common/middleware/errorHandler.middleware');
const { requestLogger } = require('./common/middleware/requestLogger.middleware');
const routes = require('./routes');

const app = express();

// Global middleware
app.use(helmet());
app.use(cors());
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));
app.use(requestLogger);

// Routes
app.use('/api', routes);

// Error handler (must be last)
app.use(errorHandler);

module.exports = app;

// server.js — starts the server
const app = require('./app');
const config = require('./config');
const logger = require('./common/utils/logger');
const { connectDB } = require('./database/pool');
const { connectRedis } = require('./config/redis');

async function startServer() {
  await connectDB();
  await connectRedis();

  app.listen(config.port, () => {
    logger.info(`Server running on port ${config.port} in ${config.env} mode`);
  });
}

startServer().catch((error) => {
  logger.error('Failed to start server', { error: error.message });
  process.exit(1);
});
```

---

## 2. App Configuration & Startup

### 2.1 Load all config in one place with validation

```js
// ❌ Bad — scattered process.env calls everywhere
// In controller:
const secret = process.env.JWT_SECRET;
// In service:
const dbUrl = process.env.DATABASE_URL;
// In middleware:
const maxAge = process.env.SESSION_MAX_AGE || 3600;

// ✅ Good — centralized config with validation

// config/index.js
const Joi = require('joi');

const envSchema = Joi.object({
  NODE_ENV: Joi.string().valid('development', 'staging', 'production').default('development'),
  PORT: Joi.number().default(3000),
  DATABASE_URL: Joi.string().required(),
  REDIS_URL: Joi.string().required(),
  JWT_SECRET: Joi.string().min(32).required(),
  JWT_EXPIRES_IN: Joi.string().default('24h'),
  LOG_LEVEL: Joi.string().valid('error', 'warn', 'info', 'debug').default('info'),
  CORS_ORIGIN: Joi.string().default('*'),
}).unknown();

const { error, value: envVars } = envSchema.validate(process.env, { abortEarly: false });

if (error) {
  throw new Error(`Config validation error: ${error.details.map(d => d.message).join(', ')}`);
}

module.exports = Object.freeze({
  env: envVars.NODE_ENV,
  port: envVars.PORT,
  databaseUrl: envVars.DATABASE_URL,
  redisUrl: envVars.REDIS_URL,
  jwt: {
    secret: envVars.JWT_SECRET,
    expiresIn: envVars.JWT_EXPIRES_IN,
  },
  logLevel: envVars.LOG_LEVEL,
  corsOrigin: envVars.CORS_ORIGIN,
});
```

---

## 3. Environment Variables

### 3.1 Use `.env` files with `.env.example`

```bash
# ❌ Bad — no .env.example, secrets in code, .env committed to git

# ✅ Good

# .env.example (committed to git — no real values)
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
REDIS_URL=redis://localhost:6379
JWT_SECRET=your-secret-key-min-32-chars-long-here
JWT_EXPIRES_IN=24h
LOG_LEVEL=info

# .gitignore (always include)
.env
.env.local
.env.production
node_modules/
logs/
```

---

## 4. Routing

### 4.1 Use Express Router with versioned, RESTful routes

```js
// ❌ Bad — all routes in app.js, no versioning, non-RESTful
app.get('/getUsers', userController.getAll);
app.post('/createUser', userController.create);
app.post('/deleteUser', userController.delete);
app.get('/getUserAppointments', appointmentController.getByUser);

// ✅ Good — modular router with versioning

// routes/index.js
const router = require('express').Router();
const userRoutes = require('../modules/user/user.routes');
const radiologistRoutes = require('../modules/radiologist/radiologist.routes');
const appointmentRoutes = require('../modules/appointment/appointment.routes');
const authRoutes = require('../modules/auth/auth.routes');

router.use('/v1/auth', authRoutes);
router.use('/v1/users', userRoutes);
router.use('/v1/radiologists', radiologistRoutes);
router.use('/v1/appointments', appointmentRoutes);

module.exports = router;

// modules/radiologist/radiologist.routes.js
const router = require('express').Router();
const controller = require('./radiologist.controller');
const { authenticate, authorize } = require('../../common/middleware/auth.middleware');
const { validate } = require('../../common/middleware/validate.middleware');
const { createSchema, updateSchema } = require('./radiologist.validation');
const asyncHandler = require('../../common/utils/asyncHandler');

router.get('/', authenticate, asyncHandler(controller.getAll));
router.get('/:id', authenticate, asyncHandler(controller.getById));
router.post('/', authenticate, authorize('admin'), validate(createSchema), asyncHandler(controller.create));
router.put('/:id', authenticate, authorize('admin'), validate(updateSchema), asyncHandler(controller.update));
router.delete('/:id', authenticate, authorize('admin'), asyncHandler(controller.remove));

module.exports = router;
```

### 4.2 Keep route files thin — only routing logic

```js
// ❌ Bad — business logic in routes
router.post('/', async (req, res) => {
  const { email, name } = req.body;
  if (!email) return res.status(400).json({ error: 'Email required' });

  const existing = await db.query('SELECT * FROM users WHERE email = $1', [email]);
  if (existing.rows.length) return res.status(409).json({ error: 'Email exists' });

  const result = await db.query('INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *', [name, email]);
  await sendWelcomeEmail(email);
  res.status(201).json(result.rows[0]);
});

// ✅ Good — route only defines path, method, middleware chain
router.post('/', authenticate, validate(createUserSchema), asyncHandler(controller.create));
```

---

## 5. Controllers

### 5.1 Controllers handle HTTP — extract request, call service, send response

```js
// ❌ Bad — controller has business logic and DB calls
const getById = async (req, res) => {
  try {
    const result = await db.query(
      'SELECT r.*, p.prefix FROM settings.radiologists r LEFT JOIN prefixes p ON r.name_prefix_uid = p.uid WHERE r.uid = $1 AND r.customer_uid = $2',
      [req.params.id, req.user.customerUid]
    );
    if (!result.rows.length) {
      return res.status(404).json({ error: 'Radiologist not found' });
    }
    const radiologist = result.rows[0];
    delete radiologist.password_hash;
    delete radiologist.internal_notes;
    res.json(radiologist);
  } catch (error) {
    console.log(error);
    res.status(500).json({ error: 'Server error' });
  }
};

// ✅ Good — controller is thin, delegates to service
const { sendSuccess } = require('../../common/utils/response');

const getById = async (req, res) => {
  const radiologist = await radiologistService.getById(req.params.id, req.user.customerUid);
  sendSuccess(res, radiologist);
};

const create = async (req, res) => {
  const radiologist = await radiologistService.create(req.body, req.user);
  sendSuccess(res, radiologist, 201);
};

const update = async (req, res) => {
  const radiologist = await radiologistService.update(req.params.id, req.body, req.user);
  sendSuccess(res, radiologist);
};

const remove = async (req, res) => {
  await radiologistService.remove(req.params.id, req.user);
  sendSuccess(res, null, 204);
};

module.exports = { getById, create, update, remove };
```

### 5.2 Never access `req` or `res` in service or repository layers

```js
// ❌ Bad — service receives req/res objects
const create = async (req, res) => {
  await userService.create(req, res); // service knows about HTTP
};

// ✅ Good — pass only the data the service needs
const create = async (req, res) => {
  const user = await userService.create(req.body, req.user.uid);
  sendSuccess(res, user, 201);
};
```

---

## 6. Services (Business Logic)

### 6.1 Services contain business rules — no HTTP, no SQL

```js
// ❌ Bad — service mixed with DB queries and HTTP concerns
class RadiologistService {
  async getById(req, res) {
    const result = await db.query('SELECT * FROM radiologists WHERE uid = $1', [req.params.id]);
    if (!result.rows.length) {
      res.status(404).json({ error: 'Not found' });
      return;
    }
    res.json(result.rows[0]);
  }
}

// ✅ Good — service handles only business logic
class RadiologistService {
  constructor(radiologistRepository, cacheService) {
    this.radiologistRepo = radiologistRepository;
    this.cache = cacheService;
  }

  async getById(id, customerUid) {
    // Check cache
    const cached = await this.cache.get(`radiologist:${id}`);
    if (cached) return cached;

    // Get from DB
    const radiologist = await this.radiologistRepo.findById(id, customerUid);
    if (!radiologist) {
      throw new NotFoundError('Radiologist');
    }

    // Cache and return
    await this.cache.set(`radiologist:${id}`, radiologist, 300);
    return radiologist;
  }

  async update(id, data, currentUser) {
    const existing = await this.radiologistRepo.findById(id, currentUser.customerUid);

    if (!existing) throw new NotFoundError('Radiologist');
    if (existing.deletedDt) throw new AppError('Radiologist has been deleted', 410);

    const updated = await this.radiologistRepo.update(id, {
      ...data,
      lastUpdatedByUid: currentUser.uid,
      lastUpdatedDt: new Date(),
    });

    // Invalidate cache
    await this.cache.del(`radiologist:${id}`);

    return updated;
  }
}

module.exports = RadiologistService;
```

---

## 7. Repository / Data Access Layer

### 7.1 Repository handles ONLY database queries

```js
// ❌ Bad — SQL scattered across controllers and services
// In controller:
await db.query('SELECT * FROM radiologists WHERE uid = $1', [id]);
// In service:
await db.query('UPDATE radiologists SET first_name = $1 WHERE uid = $2', [name, id]);
// In another controller:
await db.query('SELECT * FROM radiologists WHERE uid = $1', [id]); // duplicated

// ✅ Good — centralized in repository
class RadiologistRepository {
  constructor(pool) {
    this.pool = pool;
  }

  async findById(id, customerUid) {
    const result = await this.pool.query(
      `SELECT uid, customer_uid, first_name, last_name, middle_name,
              full_name, suffix, sourced_name, name_prefix_uid,
              created_dt, last_updated_dt
       FROM settings.radiologists
       WHERE uid = $1 AND customer_uid = $2 AND deleted_dt IS NULL`,
      [id, customerUid]
    );
    return result.rows[0] ?? null;
  }

  async findAll(customerUid, { page = 1, limit = 20, search = '' } = {}) {
    const offset = (page - 1) * limit;
    const values = [customerUid, limit, offset];
    let whereClause = 'WHERE customer_uid = $1 AND deleted_dt IS NULL';

    if (search) {
      whereClause += ' AND (first_name ILIKE $4 OR last_name ILIKE $4)';
      values.push(`%${search}%`);
    }

    const [dataResult, countResult] = await Promise.all([
      this.pool.query(
        `SELECT uid, first_name, last_name, full_name, suffix
         FROM settings.radiologists ${whereClause}
         ORDER BY last_name ASC, first_name ASC
         LIMIT $2 OFFSET $3`,
        values
      ),
      this.pool.query(
        `SELECT COUNT(*) FROM settings.radiologists ${whereClause}`,
        search ? [customerUid, `%${search}%`] : [customerUid]
      ),
    ]);

    return {
      data: dataResult.rows,
      total: parseInt(countResult.rows[0].count, 10),
    };
  }

  async create(data) {
    const result = await this.pool.query(
      `INSERT INTO settings.radiologists
        (customer_uid, name_prefix_uid, first_name, last_name, middle_name,
         full_name, suffix, sourced_name, created_by_uid, created_dt)
       VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, NOW())
       RETURNING *`,
      [
        data.customerUid, data.namePrefixUid, data.firstName,
        data.lastName, data.middleName, data.fullName,
        data.suffix, data.sourcedName, data.createdByUid,
      ]
    );
    return result.rows[0];
  }

  async update(id, data) {
    const result = await this.pool.query(
      `UPDATE settings.radiologists
       SET name_prefix_uid = $1, first_name = $2, last_name = $3,
           middle_name = $4, full_name = $5, suffix = $6,
           sourced_name = $7, last_updated_by_uid = $8, last_updated_dt = NOW()
       WHERE uid = $9 AND deleted_dt IS NULL
       RETURNING *`,
      [
        data.namePrefixUid, data.firstName, data.lastName,
        data.middleName, data.fullName, data.suffix,
        data.sourcedName, data.lastUpdatedByUid, id,
      ]
    );
    return result.rows[0] ?? null;
  }

  async softDelete(id, deletedByUid) {
    const result = await this.pool.query(
      `UPDATE settings.radiologists
       SET deleted_dt = NOW(), deleted_by_uid = $1
       WHERE uid = $2 AND deleted_dt IS NULL
       RETURNING uid`,
      [deletedByUid, id]
    );
    return result.rowCount > 0;
  }
}

module.exports = RadiologistRepository;
```

---

## 8. Middleware

### 8.1 Write focused, single-purpose middleware

```js
// ❌ Bad — one middleware doing everything
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  req.startTime = Date.now();
  if (req.headers.authorization) {
    try {
      const token = req.headers.authorization.split(' ')[1];
      req.user = jwt.verify(token, SECRET);
    } catch (e) {
      return res.status(401).json({ error: 'Invalid token' });
    }
  }
  res.on('finish', () => {
    console.log(`Response time: ${Date.now() - req.startTime}ms`);
  });
  next();
});

// ✅ Good — separate middleware for each concern

// requestLogger.middleware.js
const requestLogger = (req, res, next) => {
  const start = Date.now();

  res.on('finish', () => {
    logger.info('Request completed', {
      method: req.method,
      url: req.originalUrl,
      status: res.statusCode,
      duration: `${Date.now() - start}ms`,
      ip: req.ip,
      userAgent: req.get('user-agent'),
    });
  });

  next();
};

// auth.middleware.js
const authenticate = (req, res, next) => {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    throw new UnauthorizedError('No token provided');
  }

  try {
    const token = authHeader.split(' ')[1];
    req.user = jwt.verify(token, config.jwt.secret);
    next();
  } catch (error) {
    throw new UnauthorizedError('Invalid or expired token');
  }
};

// authorize.middleware.js
const authorize = (...allowedRoles) => (req, res, next) => {
  if (!req.user) throw new UnauthorizedError('Authentication required');
  if (!allowedRoles.includes(req.user.role)) {
    throw new ForbiddenError('Insufficient permissions');
  }
  next();
};
```

### 8.2 Middleware execution order matters

```js
// ❌ Bad — wrong order
app.use(routes);            // routes before parsing body
app.use(express.json());    // body parser too late
app.use(helmet());          // security headers after routes
app.use(errorHandler);      // this is fine

// ✅ Good — correct order
app.use(helmet());                          // 1. Security headers first
app.use(cors(corsOptions));                 // 2. CORS
app.use(express.json({ limit: '10mb' }));   // 3. Body parsing
app.use(express.urlencoded({ extended: true }));
app.use(requestLogger);                     // 4. Request logging
app.use('/api', routes);                    // 5. Routes
app.use(notFoundHandler);                   // 6. 404 handler
app.use(errorHandler);                      // 7. Error handler (always last)
```

---

## 9. Request Validation

### 9.1 Validate at the boundary using Joi or Zod

```js
// ❌ Bad — manual validation scattered in controller
const create = async (req, res) => {
  if (!req.body.firstName) return res.status(400).json({ error: 'First name required' });
  if (!req.body.lastName) return res.status(400).json({ error: 'Last name required' });
  if (req.body.firstName.length > 100) return res.status(400).json({ error: 'Name too long' });
  if (req.body.email && !req.body.email.includes('@')) return res.status(400).json({ error: 'Invalid email' });
  // ... 20 more if statements
};

// ✅ Good — schema-based validation

// radiologist.validation.js
const Joi = require('joi');

const createSchema = Joi.object({
  firstName: Joi.string().trim().min(1).max(100).required(),
  lastName: Joi.string().trim().min(1).max(100).required(),
  middleName: Joi.string().trim().max(100).allow('', null),
  suffix: Joi.string().trim().max(20).allow('', null),
  namePrefixUid: Joi.number().integer().positive().allow(null),
  sourcedName: Joi.string().trim().max(200).allow('', null),
});

const updateSchema = Joi.object({
  firstName: Joi.string().trim().min(1).max(100),
  lastName: Joi.string().trim().min(1).max(100),
  middleName: Joi.string().trim().max(100).allow('', null),
  suffix: Joi.string().trim().max(20).allow('', null),
  namePrefixUid: Joi.number().integer().positive().allow(null),
  sourcedName: Joi.string().trim().max(200).allow('', null),
}).min(1); // at least one field required for update

module.exports = { createSchema, updateSchema };

// validate.middleware.js
const validate = (schema, source = 'body') => (req, res, next) => {
  const { error, value } = schema.validate(req[source], {
    abortEarly: false,
    stripUnknown: true,
  });

  if (error) {
    const messages = error.details.map(d => d.message);
    throw new ValidationError(messages.join('; '));
  }

  req[source] = value; // use sanitized values
  next();
};

// Usage in route
router.post('/', authenticate, validate(createSchema), asyncHandler(controller.create));
router.put('/:id', authenticate, validate(updateSchema), asyncHandler(controller.update));
```

### 9.2 Validate query params and URL params too

```js
// ❌ Bad — only validating body
router.get('/:id', controller.getById); // id could be "abc" or SQL injection

// ✅ Good — validate params
const idParamSchema = Joi.object({
  id: Joi.number().integer().positive().required(),
});

const listQuerySchema = Joi.object({
  page: Joi.number().integer().min(1).default(1),
  limit: Joi.number().integer().min(1).max(100).default(20),
  search: Joi.string().trim().max(200).default(''),
  sortBy: Joi.string().valid('firstName', 'lastName', 'createdDt').default('lastName'),
  sortOrder: Joi.string().valid('asc', 'desc').default('asc'),
});

router.get('/', authenticate, validate(listQuerySchema, 'query'), asyncHandler(controller.getAll));
router.get('/:id', authenticate, validate(idParamSchema, 'params'), asyncHandler(controller.getById));
```

---

## 10. Error Handling

### 10.1 Custom error hierarchy

```js
// ❌ Bad — generic errors everywhere
throw new Error('Not found');
throw new Error('Bad request');
throw new Error('Unauthorized');

// ✅ Good — structured error classes

// common/errors/AppError.js
class AppError extends Error {
  constructor(message, statusCode, errorCode = 'UNKNOWN_ERROR') {
    super(message);
    this.name = this.constructor.name;
    this.statusCode = statusCode;
    this.errorCode = errorCode;
    this.isOperational = true;
    Error.captureStackTrace(this, this.constructor);
  }
}

class ValidationError extends AppError {
  constructor(message) { super(message, 400, 'VALIDATION_ERROR'); }
}

class NotFoundError extends AppError {
  constructor(resource = 'Resource') { super(`${resource} not found`, 404, 'NOT_FOUND'); }
}

class UnauthorizedError extends AppError {
  constructor(message = 'Authentication required') { super(message, 401, 'UNAUTHORIZED'); }
}

class ForbiddenError extends AppError {
  constructor(message = 'Insufficient permissions') { super(message, 403, 'FORBIDDEN'); }
}

class ConflictError extends AppError {
  constructor(message = 'Resource conflict') { super(message, 409, 'CONFLICT'); }
}

module.exports = { AppError, ValidationError, NotFoundError, UnauthorizedError, ForbiddenError, ConflictError };
```

### 10.2 Async handler wrapper — eliminates try/catch in every route

```js
// ❌ Bad — try/catch repeated in every controller
const getById = async (req, res) => {
  try {
    const user = await userService.getById(req.params.id);
    res.json(user);
  } catch (error) {
    if (error instanceof NotFoundError) res.status(404).json({ error: error.message });
    else res.status(500).json({ error: 'Server error' });
  }
};

// ✅ Good — asyncHandler wraps automatically
// common/utils/asyncHandler.js
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// Controller becomes clean
const getById = async (req, res) => {
  const user = await userService.getById(req.params.id);
  sendSuccess(res, user);
};

// Route
router.get('/:id', asyncHandler(getById));
```

### 10.3 Centralized error handler middleware

```js
// ✅ Good — single error handler for the entire app

// common/middleware/errorHandler.middleware.js
const errorHandler = (error, req, res, next) => {
  // Default values
  let statusCode = error.statusCode || 500;
  let message = error.message || 'Internal server error';
  let errorCode = error.errorCode || 'INTERNAL_ERROR';

  // Specific error types
  if (error.name === 'JsonWebTokenError') {
    statusCode = 401;
    message = 'Invalid token';
    errorCode = 'INVALID_TOKEN';
  }

  if (error.name === 'TokenExpiredError') {
    statusCode = 401;
    message = 'Token expired';
    errorCode = 'TOKEN_EXPIRED';
  }

  if (error.code === '23505') { // PostgreSQL unique violation
    statusCode = 409;
    message = 'Duplicate entry';
    errorCode = 'DUPLICATE';
  }

  if (error.code === '23503') { // PostgreSQL foreign key violation
    statusCode = 400;
    message = 'Referenced record not found';
    errorCode = 'FK_VIOLATION';
  }

  // Log the error
  if (statusCode >= 500) {
    logger.error('Unhandled error', {
      method: req.method,
      url: req.originalUrl,
      statusCode,
      error: error.message,
      stack: error.stack,
      body: req.body,
      params: req.params,
      userId: req.user?.uid,
    });
  } else {
    logger.warn('Client error', {
      method: req.method,
      url: req.originalUrl,
      statusCode,
      error: message,
    });
  }

  // Don't expose internal errors to client
  if (statusCode === 500 && process.env.NODE_ENV === 'production') {
    message = 'Internal server error';
  }

  res.status(statusCode).json({
    success: false,
    error: {
      message,
      code: errorCode,
      ...(process.env.NODE_ENV === 'development' && { stack: error.stack }),
    },
  });
};

module.exports = { errorHandler };
```

### 10.4 Handle unhandled rejections and uncaught exceptions

```js
// ❌ Bad — no global error handlers, app crashes silently

// ✅ Good — in server.js
process.on('unhandledRejection', (reason) => {
  logger.error('Unhandled Rejection', { error: reason?.message || reason });
  // Don't exit — let the process manager (PM2) handle restarts
});

process.on('uncaughtException', (error) => {
  logger.error('Uncaught Exception', { error: error.message, stack: error.stack });
  // Exit after logging — uncaught exceptions leave the process in an undefined state
  process.exit(1);
});
```

---

## 11. Authentication & Authorization

### 11.1 JWT authentication with proper structure

```js
// ❌ Bad — hardcoded secret, no expiry, no error handling
const token = jwt.sign({ id: user.id }, 'secret123');

// ✅ Good

// auth.service.js
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');
const config = require('../../config');

const SALT_ROUNDS = 12;

class AuthService {
  constructor(userRepository) {
    this.userRepo = userRepository;
  }

  async login(email, password) {
    const user = await this.userRepo.findByEmail(email);
    if (!user) throw new UnauthorizedError('Invalid credentials');

    const isMatch = await bcrypt.compare(password, user.passwordHash);
    if (!isMatch) throw new UnauthorizedError('Invalid credentials');

    if (!user.isActive) throw new ForbiddenError('Account is deactivated');

    const token = this.generateToken(user);
    return { token, user: this.sanitizeUser(user) };
  }

  generateToken(user) {
    return jwt.sign(
      {
        uid: user.uid,
        email: user.email,
        role: user.role,
        customerUid: user.customerUid,
      },
      config.jwt.secret,
      { expiresIn: config.jwt.expiresIn }
    );
  }

  sanitizeUser(user) {
    const { passwordHash, internalNotes, ...safeUser } = user;
    return safeUser;
  }

  async hashPassword(password) {
    return bcrypt.hash(password, SALT_ROUNDS);
  }
}

module.exports = AuthService;
```

### 11.2 Role-based authorization middleware

```js
// ❌ Bad — checking roles inside controllers
const deleteUser = async (req, res) => {
  if (req.user.role !== 'admin' && req.user.role !== 'superadmin') {
    return res.status(403).json({ error: 'Forbidden' });
  }
  // ...
};

// ✅ Good — reusable authorize middleware
const authorize = (...allowedRoles) => (req, res, next) => {
  if (!req.user) throw new UnauthorizedError();
  if (!allowedRoles.includes(req.user.role)) throw new ForbiddenError();
  next();
};

// Usage
router.delete('/:id', authenticate, authorize('admin', 'superadmin'), asyncHandler(controller.remove));
router.get('/reports', authenticate, authorize('admin', 'radiologist'), asyncHandler(controller.getReports));
```

---

## 12. Database (PostgreSQL)

### 12.1 Connection pool with proper config

```js
// ❌ Bad — new connection per query
const { Client } = require('pg');
async function query(sql, params) {
  const client = new Client({ connectionString: DATABASE_URL });
  await client.connect();
  const result = await client.query(sql, params);
  await client.end();
  return result;
}

// ✅ Good — connection pool

// database/pool.js
const { Pool } = require('pg');
const config = require('../config');
const logger = require('../common/utils/logger');

const pool = new Pool({
  connectionString: config.databaseUrl,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
});

pool.on('error', (err) => {
  logger.error('Unexpected pool error', { error: err.message });
});

pool.on('connect', () => {
  logger.debug('New database connection established');
});

const connectDB = async () => {
  try {
    const client = await pool.connect();
    logger.info('Database connected successfully');
    client.release();
  } catch (error) {
    logger.error('Database connection failed', { error: error.message });
    throw error;
  }
};

module.exports = { pool, connectDB };
```

### 12.2 Transactions with proper error handling

```js
// ✅ Good — reusable transaction helper
async function withTransaction(callback) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const result = await callback(client);
    await client.query('COMMIT');
    return result;
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}

// Usage
async function transferPatient(patientUid, fromDoctorUid, toDoctorUid, transferredByUid) {
  return withTransaction(async (client) => {
    await client.query(
      'UPDATE patients SET doctor_uid = $1, last_updated_dt = NOW() WHERE uid = $2',
      [toDoctorUid, patientUid]
    );

    await client.query(
      `INSERT INTO audit_log (entity, entity_uid, action, performed_by_uid, details, created_dt)
       VALUES ('patients', $1, 'transfer', $2, $3, NOW())`,
      [patientUid, transferredByUid, JSON.stringify({ from: fromDoctorUid, to: toDoctorUid })]
    );

    return { success: true };
  });
}
```

---

## 13. Redis Caching

### 13.1 Cache with proper invalidation strategy

```js
// ❌ Bad — cache without TTL, no invalidation
await redis.set('users', JSON.stringify(users));

// ✅ Good — structured caching utility

// common/utils/redis.js
const Redis = require('ioredis');
const config = require('../../config');
const logger = require('./logger');

const redis = new Redis(config.redisUrl);

const cacheService = {
  async get(key) {
    try {
      const data = await redis.get(key);
      return data ? JSON.parse(data) : null;
    } catch (error) {
      logger.warn('Cache get failed', { key, error: error.message });
      return null; // fail open — don't break the app if Redis is down
    }
  },

  async set(key, data, ttlSeconds = 300) {
    try {
      await redis.setex(key, ttlSeconds, JSON.stringify(data));
    } catch (error) {
      logger.warn('Cache set failed', { key, error: error.message });
    }
  },

  async del(key) {
    try {
      await redis.del(key);
    } catch (error) {
      logger.warn('Cache delete failed', { key, error: error.message });
    }
  },

  async delPattern(pattern) {
    try {
      const keys = await redis.keys(pattern);
      if (keys.length) await redis.del(...keys);
    } catch (error) {
      logger.warn('Cache pattern delete failed', { pattern, error: error.message });
    }
  },
};

module.exports = { redis, cacheService };

// Usage in service
const CACHE_KEYS = {
  radiologist: (id) => `radiologist:${id}`,
  radiologistList: (customerUid) => `radiologists:customer:${customerUid}`,
};

async function getById(id, customerUid) {
  const cached = await cacheService.get(CACHE_KEYS.radiologist(id));
  if (cached) return cached;

  const data = await radiologistRepo.findById(id, customerUid);
  if (!data) throw new NotFoundError('Radiologist');

  await cacheService.set(CACHE_KEYS.radiologist(id), data, 300);
  return data;
}

async function update(id, data) {
  const updated = await radiologistRepo.update(id, data);

  // Invalidate related caches
  await cacheService.del(CACHE_KEYS.radiologist(id));
  await cacheService.delPattern(`radiologists:customer:*`);

  return updated;
}
```

---

## 14. Logging (Winston)

### 14.1 Structured logging with context

```js
// ❌ Bad
console.log('User created');
console.log('Error:', error);
console.log(`GET /users/${id} - 200 - 45ms`);

// ✅ Good

// common/utils/logger.js
const winston = require('winston');
const config = require('../../config');

const logger = winston.createLogger({
  level: config.logLevel,
  format: winston.format.combine(
    winston.format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'ris-api' },
  transports: [
    new winston.transports.File({ filename: 'logs/error.log', level: 'error', maxsize: 10485760, maxFiles: 5 }),
    new winston.transports.File({ filename: 'logs/combined.log', maxsize: 10485760, maxFiles: 10 }),
  ],
});

if (config.env !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.combine(
      winston.format.colorize(),
      winston.format.simple()
    ),
  }));
}

module.exports = logger;

// Usage — always log with context
logger.info('Radiologist created', {
  radiologistUid: radiologist.uid,
  customerUid: currentUser.customerUid,
  createdBy: currentUser.uid,
});

logger.error('Failed to update radiologist', {
  radiologistUid: id,
  error: error.message,
  stack: error.stack,
});
```

### 14.2 Add correlation ID for request tracing

```js
// ✅ Good — track requests across services

// middleware/correlationId.middleware.js
const { v4: uuidv4 } = require('uuid');

const correlationId = (req, res, next) => {
  req.correlationId = req.headers['x-correlation-id'] || uuidv4();
  res.setHeader('x-correlation-id', req.correlationId);
  next();
};

// Use in logging
logger.info('Processing request', {
  correlationId: req.correlationId,
  method: req.method,
  url: req.originalUrl,
});
```

---

## 15. API Response Standards

### 15.1 Consistent response envelope

```js
// ❌ Bad — every endpoint returns different shapes
res.json(users);                                    // raw array
res.json({ user });                                 // object wrapper
res.json({ message: 'Created', data: user });       // different keys
res.status(400).json('Bad request');                 // string
res.status(500).json({ err: 'Something went wrong' }); // "err" not "error"

// ✅ Good — uniform response helper

// common/utils/response.js
const sendSuccess = (res, data, statusCode = 200) => {
  if (statusCode === 204) return res.status(204).end();

  res.status(statusCode).json({
    success: true,
    data,
  });
};

const sendPaginated = (res, data, pagination) => {
  res.status(200).json({
    success: true,
    data,
    pagination,
  });
};

module.exports = { sendSuccess, sendPaginated };

// All responses look consistent:
// Success:  { success: true, data: { ... } }
// Paginated: { success: true, data: [...], pagination: { page, limit, total, totalPages } }
// Error:    { success: false, error: { message, code } }
```

---

## 16. API Versioning

### 16.1 Use URL-based versioning

```js
// ❌ Bad — no versioning
app.use('/users', userRoutes);

// ❌ Bad — header-based versioning (hard to test, debug, and document)
app.use((req, res, next) => {
  const version = req.headers['api-version'];
  // ...
});

// ✅ Good — URL prefix versioning
app.use('/api/v1/users', userRoutesV1);
app.use('/api/v2/users', userRoutesV2); // when breaking changes happen

// routes/index.js
const v1Routes = require('./v1');
router.use('/v1', v1Routes);
```

---

## 17. Pagination, Sorting & Filtering

```js
// ✅ Good — reusable pagination helper

// common/utils/pagination.js
const parsePagination = (query) => {
  const page = Math.max(1, parseInt(query.page, 10) || 1);
  const limit = Math.min(100, Math.max(1, parseInt(query.limit, 10) || 20));
  const offset = (page - 1) * limit;
  return { page, limit, offset };
};

const buildPagination = (page, limit, total) => ({
  page,
  limit,
  total,
  totalPages: Math.ceil(total / limit),
  hasNextPage: page < Math.ceil(total / limit),
  hasPrevPage: page > 1,
});

module.exports = { parsePagination, buildPagination };

// Usage in controller
const getAll = async (req, res) => {
  const { page, limit, offset } = parsePagination(req.query);
  const { search, sortBy, sortOrder } = req.query;

  const { data, total } = await radiologistService.getAll(req.user.customerUid, {
    limit, offset, search, sortBy, sortOrder,
  });

  sendPaginated(res, data, buildPagination(page, limit, total));
};
```

---

## 18. File Upload Handling

```js
// ❌ Bad — no size limit, no type validation, saving to local disk
app.post('/upload', (req, res) => {
  req.pipe(fs.createWriteStream(`./uploads/${req.headers.filename}`));
  res.json({ message: 'Uploaded' });
});

// ✅ Good — multer with validation
const multer = require('multer');

const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'application/pdf'];
const MAX_FILE_SIZE = 10 * 1024 * 1024; // 10MB

const upload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: MAX_FILE_SIZE },
  fileFilter: (req, file, cb) => {
    if (!ALLOWED_TYPES.includes(file.mimetype)) {
      return cb(new ValidationError(`File type ${file.mimetype} not allowed`));
    }
    cb(null, true);
  },
});

router.post('/upload', authenticate, upload.single('file'), asyncHandler(async (req, res) => {
  if (!req.file) throw new ValidationError('No file provided');

  const result = await fileService.upload(req.file, req.user);
  sendSuccess(res, result, 201);
}));
```

---

## 19. Rate Limiting & Throttling

```js
// ✅ Good — different limits for different endpoints
const rateLimit = require('express-rate-limit');

const generalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 200,
  standardHeaders: true,
  message: { success: false, error: { message: 'Too many requests', code: 'RATE_LIMITED' } },
});

const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  message: { success: false, error: { message: 'Too many login attempts', code: 'RATE_LIMITED' } },
});

const uploadLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,
  max: 20,
  message: { success: false, error: { message: 'Upload limit reached', code: 'RATE_LIMITED' } },
});

app.use('/api', generalLimiter);
app.use('/api/v1/auth/login', authLimiter);
app.use('/api/v1/upload', uploadLimiter);
```

---

## 20. CORS & Security Headers

```js
// ❌ Bad — wide open CORS, no security headers
app.use(cors());

// ✅ Good — configured CORS + Helmet security headers
const helmet = require('helmet');
const cors = require('cors');

app.use(helmet());

const corsOptions = {
  origin: (origin, callback) => {
    const allowedOrigins = config.corsOrigin.split(',');
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new ForbiddenError('CORS not allowed'));
    }
  },
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Correlation-ID'],
  credentials: true,
  maxAge: 86400,
};

app.use(cors(corsOptions));
```

---

## 21. Health Check & Monitoring

```js
// ❌ Bad — no health check
// nothing...

// ✅ Good — health check with dependency status
router.get('/health', async (req, res) => {
  const health = {
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    version: process.env.npm_package_version || '1.0.0',
    dependencies: {},
  };

  // Check database
  try {
    await pool.query('SELECT 1');
    health.dependencies.database = 'ok';
  } catch {
    health.dependencies.database = 'error';
    health.status = 'degraded';
  }

  // Check Redis
  try {
    await redis.ping();
    health.dependencies.redis = 'ok';
  } catch {
    health.dependencies.redis = 'error';
    health.status = 'degraded';
  }

  const statusCode = health.status === 'ok' ? 200 : 503;
  res.status(statusCode).json(health);
});
```

---

## 22. Graceful Shutdown

```js
// ❌ Bad — abrupt shutdown, connections leak
// Ctrl+C just kills everything

// ✅ Good — graceful shutdown
const gracefulShutdown = async (signal) => {
  logger.info(`${signal} received. Starting graceful shutdown...`);

  // Stop accepting new requests
  server.close(async () => {
    logger.info('HTTP server closed');

    try {
      await pool.end();
      logger.info('Database pool closed');

      await redis.quit();
      logger.info('Redis connection closed');

      logger.info('Graceful shutdown complete');
      process.exit(0);
    } catch (error) {
      logger.error('Error during shutdown', { error: error.message });
      process.exit(1);
    }
  });

  // Force shutdown after 30 seconds
  setTimeout(() => {
    logger.error('Forced shutdown after timeout');
    process.exit(1);
  }, 30000);
};

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
```

---

## 23. Process Management (PM2)

```js
// ❌ Bad — running with plain node
// node server.js

// ✅ Good — PM2 ecosystem config

// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'ris-api',
    script: 'src/server.js',
    instances: 'max',
    exec_mode: 'cluster',
    autorestart: true,
    max_memory_restart: '1G',
    env: {
      NODE_ENV: 'development',
    },
    env_production: {
      NODE_ENV: 'production',
    },
    error_file: 'logs/pm2-error.log',
    out_file: 'logs/pm2-out.log',
    merge_logs: true,
    log_date_format: 'YYYY-MM-DD HH:mm:ss',
  }],
};

// Commands:
// pm2 start ecosystem.config.js --env production
// pm2 reload ris-api        # zero-downtime restart
// pm2 logs ris-api           # view logs
// pm2 monit                  # monitor dashboard
```

---

## 24. Docker

```dockerfile
# ❌ Bad Dockerfile
FROM node:20
COPY . .
RUN npm install
CMD ["node", "server.js"]
```

```dockerfile
# ✅ Good — multi-stage, non-root, optimized layers

FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force
COPY . .

FROM node:20-alpine
WORKDIR /app

# Security — run as non-root
RUN addgroup -g 1001 appgroup && \
    adduser -u 1001 -G appgroup -s /bin/sh -D appuser

COPY --from=builder --chown=appuser:appgroup /app .

USER appuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -qO- http://localhost:3000/api/health || exit 1

CMD ["node", "src/server.js"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  api:
    build: .
    ports:
      - '3000:3000'
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/ris_db
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ris_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U postgres']
      interval: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 5s
      retries: 5

volumes:
  pgdata:
```

---

## 25. Testing (Jest + Supertest)

### 25.1 Integration testing for APIs

```js
// ✅ Good — test the full request-response cycle

const request = require('supertest');
const app = require('../../app');

describe('GET /api/v1/radiologists', () => {
  let authToken;

  beforeAll(async () => {
    const res = await request(app)
      .post('/api/v1/auth/login')
      .send({ email: 'admin@test.com', password: 'password123' });
    authToken = res.body.data.token;
  });

  it('should return paginated radiologists list', async () => {
    const res = await request(app)
      .get('/api/v1/radiologists?page=1&limit=10')
      .set('Authorization', `Bearer ${authToken}`);

    expect(res.status).toBe(200);
    expect(res.body.success).toBe(true);
    expect(Array.isArray(res.body.data)).toBe(true);
    expect(res.body.pagination).toHaveProperty('page', 1);
    expect(res.body.pagination).toHaveProperty('totalPages');
  });

  it('should return 401 without auth token', async () => {
    const res = await request(app).get('/api/v1/radiologists');

    expect(res.status).toBe(401);
    expect(res.body.success).toBe(false);
  });

  it('should return 404 for non-existent radiologist', async () => {
    const res = await request(app)
      .get('/api/v1/radiologists/999999')
      .set('Authorization', `Bearer ${authToken}`);

    expect(res.status).toBe(404);
    expect(res.body.error.code).toBe('NOT_FOUND');
  });
});
```

### 25.2 Unit testing for services

```js
// ✅ Good — mock dependencies, test business logic

describe('RadiologistService', () => {
  let service;
  let mockRepo;
  let mockCache;

  beforeEach(() => {
    mockRepo = {
      findById: jest.fn(),
      create: jest.fn(),
      update: jest.fn(),
    };
    mockCache = {
      get: jest.fn().mockResolvedValue(null),
      set: jest.fn().mockResolvedValue(true),
      del: jest.fn().mockResolvedValue(true),
    };
    service = new RadiologistService(mockRepo, mockCache);
  });

  describe('getById', () => {
    it('should return cached data if available', async () => {
      const cached = { uid: 1, firstName: 'John' };
      mockCache.get.mockResolvedValue(cached);

      const result = await service.getById(1, 100);

      expect(result).toEqual(cached);
      expect(mockRepo.findById).not.toHaveBeenCalled();
    });

    it('should query DB on cache miss and cache the result', async () => {
      const dbData = { uid: 1, firstName: 'John' };
      mockRepo.findById.mockResolvedValue(dbData);

      const result = await service.getById(1, 100);

      expect(result).toEqual(dbData);
      expect(mockRepo.findById).toHaveBeenCalledWith(1, 100);
      expect(mockCache.set).toHaveBeenCalled();
    });

    it('should throw NotFoundError if radiologist does not exist', async () => {
      mockRepo.findById.mockResolvedValue(null);

      await expect(service.getById(999, 100)).rejects.toThrow(NotFoundError);
    });
  });
});
```

---

## 26. API Documentation (Swagger)

```js
// ✅ Good — auto-generated Swagger docs

const swaggerJsdoc = require('swagger-jsdoc');
const swaggerUi = require('swagger-ui-express');

const swaggerOptions = {
  definition: {
    openapi: '3.0.0',
    info: {
      title: 'RIS API',
      version: '1.0.0',
      description: 'Radiology Information System API',
    },
    servers: [
      { url: '/api/v1', description: 'Version 1' },
    ],
    components: {
      securitySchemes: {
        bearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT',
        },
      },
    },
    security: [{ bearerAuth: [] }],
  },
  apis: ['./src/modules/**/*.routes.js'],
};

const swaggerSpec = swaggerJsdoc(swaggerOptions);

// Mount in app.js (non-production only)
if (config.env !== 'production') {
  app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec));
}

// In route files — add JSDoc swagger annotations
/**
 * @swagger
 * /radiologists/{id}:
 *   get:
 *     summary: Get radiologist by ID
 *     tags: [Radiologists]
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: integer
 *     responses:
 *       200:
 *         description: Radiologist found
 *       404:
 *         description: Radiologist not found
 */
router.get('/:id', authenticate, asyncHandler(controller.getById));
```

---

## 27. Git Commit Standards

```bash
# ❌ Bad commit messages
git commit -m "fix"
git commit -m "update"
git commit -m "changes"
git commit -m "done"
git commit -m "asdfgh"

# ✅ Good — Conventional Commits
git commit -m "feat(radiologist): add bulk import endpoint"
git commit -m "fix(auth): handle expired refresh tokens correctly"
git commit -m "perf(appointment): optimize list query with index on date column"
git commit -m "refactor(user): extract validation to separate module"
git commit -m "test(radiologist): add integration tests for update endpoint"
git commit -m "docs(api): update Swagger for radiologist endpoints"
git commit -m "chore(deps): upgrade express to 4.21.0"

# Format: type(scope): description
# Types: feat, fix, perf, refactor, test, docs, chore, style, ci
```

---

## 28. Package.json Scripts

```json
{
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js",
    "test": "jest --coverage --detectOpenHandles",
    "test:watch": "jest --watch",
    "test:integration": "jest --testPathPattern=integration",
    "lint": "eslint src/",
    "lint:fix": "eslint src/ --fix",
    "format": "prettier --write 'src/**/*.js'",
    "db:migrate": "node src/database/migrate.js",
    "db:seed": "node src/database/seed.js",
    "docker:dev": "docker-compose up --build",
    "docker:prod": "docker-compose -f docker-compose.prod.yml up -d",
    "health": "curl -s http://localhost:3000/api/health | jq",
    "prepare": "husky install"
  }
}
```

---

## Quick Reference Checklist

Before every PR, verify:

- [ ] Route → Controller → Service → Repository layers respected
- [ ] No `req` or `res` objects in service or repository layer
- [ ] All endpoints have input validation (Joi/Zod)
- [ ] All DB queries use parameterized values (`$1, $2`)
- [ ] Transactions used for multi-step DB operations
- [ ] Custom error classes used (not generic `new Error()`)
- [ ] asyncHandler wraps every async route handler
- [ ] No `console.log` — Winston logger with context used
- [ ] Sensitive data stripped from API responses
- [ ] Cache invalidated after create/update/delete operations
- [ ] Proper HTTP status codes (201 for create, 204 for delete)
- [ ] Pagination on all list endpoints
- [ ] Tests cover happy path + error cases
- [ ] No hardcoded secrets — environment variables used
- [ ] Correlation ID present in log entries
- [ ] ESLint + Prettier pass with zero errors
- [ ] Commit message follows conventional commits format

---

> **Author:** Belshick Megason J B
> **Last Updated:** October 2026
> **Stack:** Node.js · Express.js · TypeScript · PostgreSQL · Redis · Docker · Jest
