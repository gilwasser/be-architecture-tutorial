# Task Management API Tutorial

This tutorial explains the architecture and implementation of a RESTful Task Management API.

## Table of Contents
- [Authentication](#authentication)
- [API Architecture](#api-architecture)
- [API Endpoints](#api-endpoints)
- [System Layers](#system-layers)
- [Implementation Examples](#implementation-examples)
- [Testing Strategy](#testing-strategy)

## Authentication

The API uses JWT (JSON Web Token) for authentication. You can use [jwt.io](https://jwt.io/) to decode and verify tokens.

Authentication Flow:
1. Frontend sends authentication request
2. Server validates and returns a token
3. Frontend includes token in subsequent requests

## API Architecture

The API follows a layered architecture pattern:

routes -> controller -> service -> accessor -> database

## API Endpoints

The API provides CRUD (Create, Read, Update, Delete) operations for tasks:

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/tasks` | Get all tasks (supports filtering, sorting, and pagination) |
| POST | `/api/tasks` | Create a new task |
| PUT | `/api/tasks/:id` | Update an existing task |
| DELETE | `/api/tasks/:id` | Delete a task |

## System Layers

### 1. Routes Layer
- Routes incoming requests to appropriate controller functions
- Handles URL parameter parsing
- Defines API endpoints

### 2. Controller Layer
Responsibilities:
- User authorization based on roles
- Request body validation
- Returns appropriate HTTP status codes
- Formats API responses

### 3. Service Layer
Responsibilities:
- Implements business logic
- Manages complex workflows
- Handles business rules and validations
- Orchestrates data operations

### 4. Accessor Layer
Responsibilities:
- Interfaces with the database
- Handles database operations
- Integrates with third-party services
- Abstracts data access patterns

### 5. Database Layer
- Stores and manages application data
- Handles data persistence
- Manages data relationships

## Implementation Examples

### Task Model Example
```javascript
// models/Task.js
const mongoose = require('mongoose');
const taskSchema = new mongoose.Schema({
    title: { type: String, required: true },
    description: { type: String },
    status: {
        type: String,
        enum: ['TODO', 'IN_PROGRESS', 'COMPLETED'],
        default: 'TODO'
    },
    assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
    dueDate: { type: Date },
    createdAt: { type: Date, default: Date.now },
    updatedAt: { type: Date, default: Date.now }
});
```

### Route Example
```javascript
// routes/tasks.js
const express = require('express');
const router = express.Router();
const TaskController = require('../controllers/TaskController');
const authMiddleware = require('../middleware/auth');

router.get('/', authMiddleware, TaskController.getAllTasks);
router.post('/', authMiddleware, TaskController.createTask);
router.put('/:id', authMiddleware, TaskController.updateTask);
router.delete('/:id', authMiddleware, TaskController.deleteTask);
```

### Controller Example
```javascript
// controllers/TaskController.js
const TaskService = require('../services/TaskService');
const { validateTask } = require('../validators/taskValidator');

class TaskController {
    async getAllTasks(req, res) {
        try {
            const { page = 1, limit = 10, status, sortBy } = req.query;
            const filters = status ? { status } : {};
            
            const tasks = await TaskService.getTasks(filters, {
                page: parseInt(page),
                limit: parseInt(limit),
                sortBy
            });
            
            res.status(200).json(tasks);
        } catch (error) {
            res.status(500).json({ error: 'Failed to fetch tasks' });
        }
    }

    async createTask(req, res) {
        try {
            const { error } = validateTask(req.body);
            if (error) {
                return res.status(400).json({ error: error.details[0].message });
            }

            const task = await TaskService.createTask({
                ...req.body,
                createdBy: req.user.id
            });
            
            res.status(201).json(task);
        } catch (error) {
            res.status(500).json({ error: 'Failed to create task' });
        }
    }

    async updateTask(req, res) {
        try {
            const { id } = req.params;
            const { error } = validateTask(req.body, true);
            if (error) {
                return res.status(400).json({ error: error.details[0].message });
            }

            const task = await TaskService.updateTask(id, req.body);
            if (!task) {
                return res.status(404).json({ error: 'Task not found' });
            }

            res.status(200).json(task);
        } catch (error) {
            res.status(500).json({ error: 'Failed to update task' });
        }
    }

    async deleteTask(req, res) {
        try {
            const { id } = req.params;
            const deleted = await TaskService.deleteTask(id);
            
            if (!deleted) {
                return res.status(404).json({ error: 'Task not found' });
            }

            res.status(200).json({ message: 'Task deleted successfully' });
        } catch (error) {
            res.status(500).json({ error: 'Failed to delete task' });
        }
    }
}

module.exports = new TaskController();
```

### Service Example
```javascript
// services/TaskService.js
const TaskAccessor = require('../accessors/TaskAccessor');
const { NotFoundError, ValidationError } = require('../utils/errors');

class TaskService {
    async getTasks(filters = {}, options = {}) {
        try {
            return await TaskAccessor.findTasks(filters, options);
        } catch (error) {
            throw new Error('Error fetching tasks: ' + error.message);
        }
    }

    async createTask(taskData) {
        try {
            // Additional business logic/validation can go here
            if (taskData.dueDate && new Date(taskData.dueDate) < new Date()) {
                throw new ValidationError('Due date cannot be in the past');
            }

            // You might want to check user permissions here
            return await TaskAccessor.createTask(taskData);
        } catch (error) {
            if (error instanceof ValidationError) {
                throw error;
            }
            throw new Error('Error creating task: ' + error.message);
        }
    }

    async updateTask(taskId, updateData) {
        try {
            const task = await TaskAccessor.findTaskById(taskId);
            if (!task) {
                throw new NotFoundError('Task not found');
            }

            // Business logic for status transitions
            if (updateData.status && task.status !== updateData.status) {
                this.validateStatusTransition(task.status, updateData.status);
            }

            return await TaskAccessor.updateTask(taskId, updateData);
        } catch (error) {
            if (error instanceof NotFoundError) {
                throw error;
            }
            throw new Error('Error updating task: ' + error.message);
        }
    }

    async deleteTask(taskId) {
        try {
            const task = await TaskAccessor.findTaskById(taskId);
            if (!task) {
                throw new NotFoundError('Task not found');
            }

            return await TaskAccessor.deleteTask(taskId);
        } catch (error) {
            if (error instanceof NotFoundError) {
                throw error;
            }
            throw new Error('Error deleting task: ' + error.message);
        }
    }

    validateStatusTransition(currentStatus, newStatus) {
        const validTransitions = {
            'TODO': ['IN_PROGRESS'],
            'IN_PROGRESS': ['COMPLETED', 'TODO'],
            'COMPLETED': ['IN_PROGRESS']
        };

        if (!validTransitions[currentStatus].includes(newStatus)) {
            throw new ValidationError(`Invalid status transition from ${currentStatus} to ${newStatus}`);
        }
    }
}

module.exports = new TaskService();
```

### Accessor Example
```javascript
// accessors/TaskAccessor.js
const Task = require('../models/Task');

class TaskAccessor {
    async findTasks(filters = {}, options = {}) {
        const { page = 1, limit = 10, sortBy = 'createdAt' } = options;
        const skip = (page - 1) * limit;

        const query = Task.find(filters)
            .skip(skip)
            .limit(limit)
            .sort(sortBy);

        // If we need to populate any references
        if (options.populate) {
            query.populate('assignedTo', 'name email');
        }

        return await query.exec();
    }

    async findTaskById(taskId) {
        return await Task.findById(taskId).exec();
    }

    async createTask(taskData) {
        const task = new Task(taskData);
        return await task.save();
    }

    async updateTask(taskId, updateData) {
        return await Task.findByIdAndUpdate(
            taskId,
            { ...updateData, updatedAt: Date.now() },
            { new: true, runValidators: true }
        ).exec();
    }

    async deleteTask(taskId) {
        return await Task.findByIdAndDelete(taskId).exec();
    }

    async countTasks(filters = {}) {
        return await Task.countDocuments(filters).exec();
    }

    async findTasksByUser(userId, status) {
        const query = { assignedTo: userId };
        if (status) {
            query.status = status;
        }
        return await Task.find(query).sort('-createdAt').exec();
    }
}

module.exports = new TaskAccessor();
```

## Testing Strategy

### e2e tests supertests
Each layer should have its own unit tests:
End-to-end (E2E) testing using Supertest is a powerful approach that validates the entire request-response cycle of your API routes. While many developers prefer unit testing for its granular coverage and isolation, E2E tests provide confidence that your integrated components work together as expected. With Supertest, you can simulate HTTP requests and verify the complete flow from route handling through to database operations, ensuring your API behaves correctly from the client's perspective.


## Testing Strategy

### Unit Testing
Each layer should have its own unit tests:

1. Model Tests:
- Validate schema requirements
- Test custom methods and validations

2. Controller Tests:
- Test request handling
- Verify response formats
- Test error handling

3. Service Tests:
- Test business logic
- Verify workflow handling
- Test edge cases

4. Accessor Tests:
- Test database operations
- Verify data persistence
- Test third-party service interactions


### Example Test
```javascript
// tests/services/TaskService.test.js
describe('TaskService', () => {
    describe('createTask', () => {
        it('should create a new task with valid data', async () => {
            const taskData = {
                title: 'Test Task',
                description: 'Test Description'
            };
            const task = await TaskService.createTask(taskData);
            expect(task.title).toBe(taskData.title);
            expect(task.status).toBe('TODO');
        });
    });
});
```

### E2E Tests with Supertest
```javascript
// tests/e2e/tasks.test.js
const request = require('supertest');
const app = require('../../app');
const mongoose = require('mongoose');
const { Task } = require('../../models/Task');
const { generateAuthToken } = require('../../utils/auth');

describe('Tasks API E2E Tests', () => {
    let authToken;

    beforeAll(async () => {
        authToken = generateAuthToken({ id: 'testUserId', role: 'user' });
    });

    beforeEach(async () => {
        await Task.deleteMany({}); // Clean up before each test
    });

    afterAll(async () => {
        await mongoose.connection.close();
    });

    describe('GET /api/tasks', () => {
        it('should return all tasks', async () => {
            // Create test tasks
            await Task.create([
                { title: 'Task 1', description: 'Description 1' },
                { title: 'Task 2', description: 'Description 2' }
            ]);

            const response = await request(app)
                .get('/api/tasks')
                .set('Authorization', `Bearer ${authToken}`);

            expect(response.status).toBe(200);
            expect(response.body.length).toBe(2);
            expect(response.body[0]).toHaveProperty('title', 'Task 1');
        });
    });
});

## Best Practices

### Error Handling
- Use try-catch blocks in async functions
- Create custom error classes
- Return appropriate HTTP status codes
- Provide meaningful error messages

### Security
- Validate and sanitize all input
- Use HTTPS
- Implement rate limiting
- Keep dependencies updated
- Use environment variables for sensitive data

### Performance
- Implement pagination for list endpoints
- Use database indexes
- Cache frequently accessed data
- Optimize database queries

### Documentation
- Document all API endpoints
- Include request/response examples
- Document error responses
- Keep documentation up-to-date

## Conclusion

This API architecture provides a scalable and maintainable solution for task management. By following the layered approach and implementing proper testing and security measures, you can build a robust and reliable system.

Remember to:
- Keep concerns separated between layers
- Write comprehensive tests
- Handle errors gracefully
- Document your code and API
- Follow security best practices
