# Building a CRUD Web Application with NodeJs and SQLite

This is a great combination for creating lightweight yet powerful applications that can perform Create, Read, Update, and Delete operations.

---

## Features of this CRUD Application

This task management application includes:

1. **Create** - Add new tasks with a title, description, and status
2. **Read** - View a list of all tasks and individual task details
3. **Update** - Edit existing tasks
4. **Delete** - Remove tasks

---

## How the Application Works

1. **Express** serves as the web framework
2. **SQLite** provides a lightweight database that's stored in a file
3. **EJS** templates render the HTML views
4. **method-override** enables PUT and DELETE HTTP methods from forms

The application follows the MVC (Model-View-Controller) pattern:

- **Models**: Handle database operations
- **Views**: Display data to the user
- **Routes**: Control the application flow

---

## Getting Started

### 1. Initialize your project:

```bash
mkdir nodejs-simple-crud
cd nodejs-simple-crud
npm init -y
```

### 2. Install dependencies:

```bash
npm install express ejs sqlite3 method-override body-parser
npm install nodemon --save-dev
```

### 3. Create the Project Structure

```
/nodejs-simple-crud
├── app.js                  // Main application file
├── package.json            // Project dependencies
├── database.js             // Database setup and connection
├── routes/
│   └── tasks.js            // CRUD routes for tasks resource
├── models/
│   └── task.js             // Task model with database operations
├── public/                 // Static files
│   ├── css/
│   │   └── styles.css      // Application styles
│   └── js/
│       └── main.js         // Client-side JavaScript
└── views/
    ├── layout.ejs          // Main layout template
    ├── index.ejs           // Main page with task list
    ├── create.ejs          // Form to create new task
    └── edit.ejs            // Form to edit existing task
```

### 4. Implement Code & Logic

#### Step-1: Add Project dependencies (`package.json`)

```json
// package.json
{
  "name": "nodejs-sqlite-nodejs-simple-crud",
  "version": "1.0.0",
  "description": "A simple CRUD application using Node.js and SQLite",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "dev": "nodemon app.js"
  },
  "dependencies": {
    "ejs": "^3.1.9",
    "express": "^4.18.2",
    "sqlite3": "^5.1.6",
    "method-override": "^3.0.0",
    "body-parser": "^1.20.2"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

#### Step-2: Set up database connection (`database.js`)

```javascript
// database.js
const sqlite3 = require('sqlite3').verbose();
const path = require('path');

// Connect to SQLite database
const dbPath = path.resolve(\_\_dirname, 'tasks.db');
const db = new sqlite3.Database(dbPath, (err) => {
  if (err) {
    console.error('Error connecting to database:', err.message);
    return;
  }
  console.log('Connected to the SQLite database.');

  // Create tasks table if it doesn't exist
  db.run(`CREATE TABLE IF NOT EXISTS tasks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    description TEXT,
    status TEXT DEFAULT 'pending',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
  )`, (err) => {
    if (err) {
      console.error('Error creating table:', err.message);
    } else {
      console.log('Tasks table ready.');
    }
  });
});

module.exports = db;
```

#### Step-3: Create Task model (`models/task.js`)

```javascript
// models/task.js
const db = require("../database");

class Task {
  // Get all tasks
  static getAll(callback) {
    const sql = "SELECT * FROM tasks ORDER BY created_at DESC";
    db.all(sql, [], (err, rows) => {
      if (err) {
        return callback(err, null);
      }
      return callback(null, rows);
    });
  }

  // Get a single task by ID
  static getById(id, callback) {
    const sql = "SELECT * FROM tasks WHERE id = ?";
    db.get(sql, [id], (err, row) => {
      if (err) {
        return callback(err, null);
      }
      return callback(null, row);
    });
  }

  // Create a new task
  static create(taskData, callback) {
    const { title, description, status } = taskData;
    const sql =
      "INSERT INTO tasks (title, description, status) VALUES (?, ?, ?)";
    db.run(sql, [title, description, status || "pending"], function (err) {
      if (err) {
        return callback(err, null);
      }
      return callback(null, this.lastID);
    });
  }

  // Update an existing task
  static update(id, taskData, callback) {
    const { title, description, status } = taskData;
    const sql =
      "UPDATE tasks SET title = ?, description = ?, status = ? WHERE id = ?";
    db.run(sql, [title, description, status, id], function (err) {
      if (err) {
        return callback(err, null);
      }
      return callback(null, this.changes);
    });
  }

  // Delete a task
  static delete(id, callback) {
    const sql = "DELETE FROM tasks WHERE id = ?";
    db.run(sql, [id], function (err) {
      if (err) {
        return callback(err, null);
      }
      return callback(null, this.changes);
    });
  }
}

module.exports = Task;
```

#### Step-4: Create Routes (`routes/tasks.js`)

```javascript
// routes/tasks.js
const express = require("express");
const router = express.Router();
const Task = require("../models/task");

// List all tasks
router.get("/", (req, res) => {
  Task.getAll((err, tasks) => {
    if (err) {
      return res.status(500).send("Error retrieving tasks");
    }
    res.render("index", { tasks });
  });
});

// Show form to create a new task
router.get("/new", (req, res) => {
  res.render("create");
});

// Create a new task
router.post("/", (req, res) => {
  const taskData = {
    title: req.body.title,
    description: req.body.description,
    status: req.body.status,
  };

  Task.create(taskData, (err, taskId) => {
    if (err) {
      return res.status(500).send("Error creating task");
    }
    res.redirect("/tasks");
  });
});

// Show form to edit a task
router.get("/:id/edit", (req, res) => {
  Task.getById(req.params.id, (err, task) => {
    if (err || !task) {
      return res.status(404).send("Task not found");
    }
    res.render("edit", { task });
  });
});

// Update a task
router.put("/:id", (req, res) => {
  const taskData = {
    title: req.body.title,
    description: req.body.description,
    status: req.body.status,
  };

  Task.update(req.params.id, taskData, (err, changes) => {
    if (err) {
      return res.status(500).send("Error updating task");
    }
    res.redirect("/tasks");
  });
});

// Delete a task
router.delete("/:id", (req, res) => {
  Task.delete(req.params.id, (err, changes) => {
    if (err) {
      return res.status(500).send("Error deleting task");
    }
    res.redirect("/tasks");
  });
});

// Show a single task
router.get("/:id", (req, res) => {
  Task.getById(req.params.id, (err, task) => {
    if (err || !task) {
      return res.status(404).send("Task not found");
    }
    res.render("show", { task });
  });
});

module.exports = router;
```

#### Step-5: Create Main app file (`app.js`)

```javascript
// app.js
const express = require('express');
const bodyParser = require('body-parser');
const methodOverride = require('method-override');
const path = require('path');
const app = express();
const port = process.env.PORT || 3000;

// Set up EJS as the view engine
app.set('view engine', 'ejs');
app.set('views', path.join(\_\_dirname, 'views'));

// Middleware
app.use(express.static(path.join(\_\_dirname, 'public')));
app.use(bodyParser.urlencoded({ extended: false }));
app.use(methodOverride('\_method')); // For PUT and DELETE methods in forms

// Routes
const tasksRoutes = require('./routes/tasks');
app.use('/tasks', tasksRoutes);

// Redirect root to tasks
app.get('/', (req, res) => {
  res.redirect('/tasks');
});

// Start the server
app.listen(port, () => {
  console.log(`Server running on http://localhost:${port}`);
});
```

#### Step-6: Create view templates

##### `views/layout.ejs`

```html
// views/layout.ejs

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Task Manager</title>
    <link rel="stylesheet" href="/css/styles.css" />
  </head>
  <body>
    <header>
      <nav>
        <div class="logo">Task Manager</div>
        <ul>
          <li><a href="/tasks">All Tasks</a></li>
          <li><a href="/tasks/new">Add Task</a></li>
        </ul>
      </nav>
    </header>

    <main><%- body %></main>

    <footer>
      <p>&copy; 2025 Task Manager App</p>
    </footer>

    <script src="/js/main.js"></script>
  </body>
</html>
```

##### `views/index.ejs`

```html
// views/index.ejs <%- include('layout') %>

<div class="container">
  <h1>Task List</h1>

  <% if (tasks.length === 0) { %>
  <p>No tasks found. <a href="/tasks/new">Create a new task</a>.</p>
  <% } else { %>
  <div class="task-list">
    <% tasks.forEach(task => { %>
    <div class="task-item">
      <h3><%= task.title %></h3>
      <p><%= task.description %></p>
      <div class="task-meta">
        <span class="status <%= task.status %>"><%= task.status %></span>
        <span class="date"
          >Created: <%= new Date(task.created_at).toLocaleString() %></span
        >
      </div>
      <div class="task-actions">
        <a href="/tasks/<%= task.id %>" class="btn btn-view">View</a>
        <a href="/tasks/<%= task.id %>/edit" class="btn btn-edit">Edit</a>
        <form
          action="/tasks/<%= task.id %>?_method=DELETE"
          method="POST"
          class="delete-form">
          <button
            type="submit"
            class="btn btn-delete"
            onclick="return confirm('Are you sure you want to delete this task?')">
            Delete
          </button>
        </form>
      </div>
    </div>
    <% }) %>
  </div>
  <% } %>
</div>
```

##### `views/create.ejs`

```html
// views/create.ejs <%- include('layout') %>

<div class="container">
  <h1>Create New Task</h1>

  <form action="/tasks" method="POST" class="task-form">
    <div class="form-group">
      <label for="title">Title</label>
      <input type="text" id="title" name="title" required />
    </div>

    <div class="form-group">
      <label for="description">Description</label>
      <textarea id="description" name="description" rows="5"></textarea>
    </div>

    <div class="form-group">
      <label for="status">Status</label>
      <select id="status" name="status">
        <option value="pending">Pending</option>
        <option value="in-progress">In Progress</option>
        <option value="completed">Completed</option>
      </select>
    </div>

    <div class="form-actions">
      <button type="submit" class="btn btn-primary">Create Task</button>
      <a href="/tasks" class="btn btn-secondary">Cancel</a>
    </div>
  </form>
</div>
```

##### `views/edit.ejs`

```html
// views/edit.ejs
<%- include('layout') %>

<div class="container">
  <h1>Edit Task</h1>

  <form action="/tasks/<%= task.id %>?_method=PUT" method="POST" class="task-form">
    <div class="form-group">
      <label for="title">Title</label>
      <input type="text" id="title" name="title" value="<%= task.title %>" required>
    </div>

    <div class="form-group">
      <label for="description">Description</label>
      <textarea id="description" name="description" rows="5"><%= task.description %></textarea>
    </div>

    <div class="form-group">
      <label for="status">Status</label>
      <select id="status" name="status">
        <option value="pending" <%= task.status === 'pending' ? 'selected' : '' %>>Pending</option>
        <option value="in-progress" <%= task.status === 'in-progress' ? 'selected' : '' %>>In Progress</option>
        <option value="completed" <%= task.status === 'completed' ? 'selected' : '' %>>Completed</option>
      </select>
    </div>

    <div class="form-actions">
      <button type="submit" class="btn btn-primary">Update Task</button>
      <a href="/tasks" class="btn btn-secondary">Cancel</a>
    </div>
  </form>
</div>
```

##### `views/show.ejs`

```html
// views/show.ejs <%- include('layout') %>

<div class="container">
  <div class="task-details">
    <h1><%= task.title %></h1>

    <div class="task-meta">
      <span class="status <%= task.status %>"><%= task.status %></span>
      <span class="date"
        >Created: <%= new Date(task.created_at).toLocaleString() %></span
      >
    </div>

    <div class="task-description">
      <h3>Description</h3>
      <p><%= task.description %></p>
    </div>

    <div class="task-actions">
      <a href="/tasks/<%= task.id %>/edit" class="btn btn-edit">Edit</a>
      <form
        action="/tasks/<%= task.id %>?_method=DELETE"
        method="POST"
        class="delete-form">
        <button
          type="submit"
          class="btn btn-delete"
          onclick="return confirm('Are you sure you want to delete this task?')">
          Delete
        </button>
      </form>
      <a href="/tasks" class="btn btn-secondary">Back to List</a>
    </div>
  </div>
</div>
```

#### Step-7: Add some basic CSS

##### `public/css/styles.css`

```css
// public/css/styles.css

- {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  line-height: 1.6;
  color: #333;
  background-color: #f9f9f9;
  padding-bottom: 60px; /_ For footer space _/
}

header {
  background-color: #007bff;
  color: #fff;
  padding: 1rem 0;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

nav {
  display: flex;
  justify-content: space-between;
  align-items: center;
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 2rem;
}

.logo {
  font-size: 1.5rem;
  font-weight: bold;
}

nav ul {
  display: flex;
  list-style: none;
}

nav ul li {
  margin-left: 1.5rem;
}

nav a {
  color: #fff;
  text-decoration: none;
  transition: color 0.3s;
}

nav a:hover {
  color: #f0f0f0;
}

main {
  max-width: 1200px;
  margin: 2rem auto;
  padding: 0 2rem;
}

.container {
  background-color: #fff;
  border-radius: 5px;
  padding: 2rem;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.08);
}

h1 {
  margin-bottom: 1.5rem;
  color: #333;
}

/_ Task list styling _/
.task-list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 1.5rem;
}

.task-item {
  background-color: #fff;
  border: 1px solid #e0e0e0;
  border-radius: 5px;
  padding: 1.5rem;
  transition: transform 0.3s, box-shadow 0.3s;
}

.task-item:hover {
  transform: translateY(-5px);
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.1);
}

.task-item h3 {
  margin-bottom: 0.5rem;
  color: #007bff;
}

.task-meta {
  margin: 1rem 0;
  font-size: 0.9rem;
  display: flex;
  justify-content: space-between;
}

.status {
  padding: 0.25rem 0.5rem;
  border-radius: 3px;
  font-weight: bold;
  text-transform: capitalize;
}

.status.pending {
  background-color: #ffd166;
  color: #806000;
}

.status.in-progress {
  background-color: #06d6a0;
  color: #004d3d;
}

.status.completed {
  background-color: #118ab2;
  color: #fff;
}

.date {
  color: #777;
}

.task-actions {
  display: flex;
  gap: 0.5rem;
  margin-top: 1rem;
}

/_ Button styling _/
.btn {
  display: inline-block;
  padding: 0.5rem 0.75rem;
  background-color: #e0e0e0;
  color: #333;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  text-decoration: none;
  font-size: 0.9rem;
  transition: background-color 0.3s;
}

.btn-primary {
  background-color: #007bff;
  color: white;
}

.btn-secondary {
  background-color: #6c757d;
  color: white;
}

.btn-view {
  background-color: #17a2b8;
  color: white;
}

.btn-edit {
  background-color: #ffc107;
  color: #333;
}

.btn-delete {
  background-color: #dc3545;
  color: white;
}

.btn:hover {
  opacity: 0.9;
}

/_ Form styling _/
.task-form {
  max-width: 600px;
}

.form-group {
  margin-bottom: 1.5rem;
}

.form-group label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: bold;
}

.form-group input,
.form-group textarea,
.form-group select {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-family: inherit;
  font-size: 1rem;
}

.form-actions {
  display: flex;
  gap: 1rem;
}

/_ Task detail view _/
  .task-details h1 {
  color: #007bff;
}

.task-description {
  margin: 1.5rem 0;
  line-height: 1.8;
}

.task-description h3 {
  margin-bottom: 0.75rem;
  color: #333;
}

footer {
  position: fixed;
  bottom: 0;
  width: 100%;
  background-color: #343a40;
  color: #fff;
  text-align: center;
  padding: 1rem 0;
}
```

#### Step-8: Add some basic client-side JavaScript

##### `public/js/main.js`

```javascript
// public/js/main.js
document.addEventListener('DOMContentLoaded', () => {
  // Add any client-side interactive features here
  console.log('Task Manager application loaded');

  // Fade in tasks on page load
  const taskItems = document.querySelectorAll('.task-item');
  taskItems.forEach((item, index) => {
    item.style.opacity = '0';
    setTimeout(() => {
      item.style.transition = 'opacity 0.5s ease-in-out';
      item.style.opacity = '1';
    }, index \* 100);
  });
});
```

### 5. Start the application

```bash
npm run dev
```

### 6. Access the application

In your browser at http://localhost:3000

---

## Customization Options

You can easily expand this application by:

1. Adding user authentication
2. Implementing categories or tags for tasks
3. Adding a due date feature
4. Creating a search functionality
5. Adding pagination for the task list

---
