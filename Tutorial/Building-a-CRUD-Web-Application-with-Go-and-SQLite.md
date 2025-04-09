# Building a CRUD Web Application with Go and SQLite

This is step-by-step guide for building a complete CRUD web application using Go and SQLite. This tutorial walks you through creating a simple task management system with both a RESTful API and an optional web interface.

Here's what you'll learn:

## Key Components

Project Setup - Creating your Go module and directory structure
SQLite Database Integration - Using GORM with SQLite for data persistence
API Development - Building RESTful endpoints for all CRUD operations:

## Create new tasks
Read tasks (all or by ID)
Update existing tasks
Delete tasks


Web Interface - A simple HTML/JavaScript frontend to interact with your API

Technical Stack

Go - For the backend application
SQLite - For database storage (lightweight, no installation required)
GORM - For object-relational mapping
Gorilla Mux - For HTTP routing

Features

Complete RESTful API
JSON response formatting
Error handling
Clean project structure
Optional web interface
Data persistence with SQLite

The guide includes all the code you need, from model definitions to API handlers and the main application file. It also provides curl commands for testing your API endpoints and instructions for adding a simple web interface.
Would you like me to explain any particular part of the implementation in more detail?



# Building a CRUD Web Application with Go and SQLite

This tutorial will guide you through creating a complete CRUD (Create, Read, Update, Delete) web application using Go and SQLite. We'll build a RESTful API for managing a simple task manager.

## Prerequisites

- Go installed on your machine (version 1.16+ recommended)
- Basic knowledge of Go syntax
- Basic understanding of RESTful APIs
- A text editor or IDE

## Step 1: Project Setup

First, let's create a new directory for our project and initialize it as a Go module:

```bash
mkdir go-sqlite-crud
cd go-sqlite-crud
go mod init github.com/yourusername/go-sqlite-crud
```

## Step 2: Installing Dependencies

We'll need the following packages:

```bash
go get github.com/gorilla/mux          # For routing
go get github.com/mattn/go-sqlite3     # SQLite driver
go get gorm.io/gorm                    # ORM library
go get gorm.io/driver/sqlite           # GORM SQLite driver
```

## Step 3: Create Project Structure

Let's organize our application with a clear structure:

```
go-sqlite-crud/
├── main.go          # Entry point
├── handlers/        # HTTP handlers
│   └── tasks.go
├── models/          # Data models
│   └── task.go
├── database/        # Database connection
│   └── database.go
└── go.mod           # Go module file
```

Create these directories:
```bash
mkdir handlers models database
```

## Step 4: Define the Task Model

Create `models/task.go`:

```go
package models

import (
    "time"

    "gorm.io/gorm"
)

// Task represents a task entity in our application
type Task struct {
    ID          uint      `json:"id" gorm:"primaryKey"`
    Title       string    `json:"title"`
    Description string    `json:"description"`
    Status      string    `json:"status"`
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at"`
}
```

## Step 5: Set Up Database Connection

Create `database/database.go`:

```go
package database

import (
    "fmt"
    "log"

    "github.com/yourusername/go-sqlite-crud/models"
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

var DB *gorm.DB

// ConnectDB connects to the SQLite database
func ConnectDB() {
    var err error
    
    // Open a connection to an SQLite database file named "tasks.db"
    // If the file doesn't exist, it will be created
    DB, err = gorm.Open(sqlite.Open("tasks.db"), &gorm.Config{})
    if err != nil {
        log.Fatal("Failed to connect to database:", err)
    }

    fmt.Println("Connected to SQLite database successfully")

    // Auto Migrate the Task model to the database
    err = DB.AutoMigrate(&models.Task{})
    if err != nil {
        log.Fatal("Failed to migrate database:", err)
    }
    
    fmt.Println("Database migration completed")
}
```

## Step 6: Implement Task Handlers

Create `handlers/tasks.go`:

```go
package handlers

import (
    "encoding/json"
    "net/http"

    "github.com/gorilla/mux"
    "github.com/yourusername/go-sqlite-crud/database"
    "github.com/yourusername/go-sqlite-crud/models"
)

// Response is a generic response struct
type Response struct {
    Status  int         `json:"status"`
    Message string      `json:"message"`
    Data    interface{} `json:"data,omitempty"`
}

// GetAllTasks returns all tasks
func GetAllTasks(w http.ResponseWriter, r *http.Request) {
    var tasks []models.Task
    
    result := database.DB.Find(&tasks)
    if result.Error != nil {
        respondWithError(w, http.StatusInternalServerError, "Error retrieving tasks")
        return
    }

    respondWithJSON(w, http.StatusOK, Response{
        Status:  http.StatusOK,
        Message: "Tasks retrieved successfully",
        Data:    tasks,
    })
}

// GetTask returns a single task by ID
func GetTask(w http.ResponseWriter, r *http.Request) {
    params := mux.Vars(r)
    var task models.Task

    result := database.DB.First(&task, params["id"])
    if result.Error != nil {
        respondWithError(w, http.StatusNotFound, "Task not found")
        return
    }

    respondWithJSON(w, http.StatusOK, Response{
        Status:  http.StatusOK,
        Message: "Task retrieved successfully",
        Data:    task,
    })
}

// CreateTask creates a new task
func CreateTask(w http.ResponseWriter, r *http.Request) {
    var task models.Task
    decoder := json.NewDecoder(r.Body)
    
    if err := decoder.Decode(&task); err != nil {
        respondWithError(w, http.StatusBadRequest, "Invalid request payload")
        return
    }
    defer r.Body.Close()

    result := database.DB.Create(&task)
    if result.Error != nil {
        respondWithError(w, http.StatusInternalServerError, "Error creating task")
        return
    }

    respondWithJSON(w, http.StatusCreated, Response{
        Status:  http.StatusCreated,
        Message: "Task created successfully",
        Data:    task,
    })
}

// UpdateTask updates an existing task
func UpdateTask(w http.ResponseWriter, r *http.Request) {
    params := mux.Vars(r)
    var task models.Task

    // Check if task exists
    result := database.DB.First(&task, params["id"])
    if result.Error != nil {
        respondWithError(w, http.StatusNotFound, "Task not found")
        return
    }

    // Decode the updated task
    var updatedTask models.Task
    decoder := json.NewDecoder(r.Body)
    if err := decoder.Decode(&updatedTask); err != nil {
        respondWithError(w, http.StatusBadRequest, "Invalid request payload")
        return
    }
    defer r.Body.Close()

    // Update fields
    task.Title = updatedTask.Title
    task.Description = updatedTask.Description
    task.Status = updatedTask.Status

    // Save changes
    database.DB.Save(&task)

    respondWithJSON(w, http.StatusOK, Response{
        Status:  http.StatusOK,
        Message: "Task updated successfully",
        Data:    task,
    })
}

// DeleteTask deletes a task by ID
func DeleteTask(w http.ResponseWriter, r *http.Request) {
    params := mux.Vars(r)
    var task models.Task

    // Check if task exists
    result := database.DB.First(&task, params["id"])
    if result.Error != nil {
        respondWithError(w, http.StatusNotFound, "Task not found")
        return
    }

    // Delete the task
    database.DB.Delete(&task)

    respondWithJSON(w, http.StatusOK, Response{
        Status:  http.StatusOK,
        Message: "Task deleted successfully",
    })
}

// Helper function to respond with JSON
func respondWithJSON(w http.ResponseWriter, code int, payload interface{}) {
    response, _ := json.Marshal(payload)
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(code)
    w.Write(response)
}

// Helper function to respond with an error
func respondWithError(w http.ResponseWriter, code int, message string) {
    respondWithJSON(w, code, Response{
        Status:  code,
        Message: message,
    })
}
```

## Step 7: Create Main Application File

Create `main.go`:

```go
package main

import (
    "fmt"
    "log"
    "net/http"

    "github.com/gorilla/mux"
    "github.com/yourusername/go-sqlite-crud/database"
    "github.com/yourusername/go-sqlite-crud/handlers"
)

func main() {
    // Connect to database
    database.ConnectDB()

    // Initialize router
    router := mux.NewRouter()

    // Routes
    router.HandleFunc("/api/tasks", handlers.GetAllTasks).Methods("GET")
    router.HandleFunc("/api/tasks/{id}", handlers.GetTask).Methods("GET")
    router.HandleFunc("/api/tasks", handlers.CreateTask).Methods("POST")
    router.HandleFunc("/api/tasks/{id}", handlers.UpdateTask).Methods("PUT")
    router.HandleFunc("/api/tasks/{id}", handlers.DeleteTask).Methods("DELETE")

    // Start server
    port := ":8080"
    fmt.Printf("Server is running on http://localhost%s\n", port)
    log.Fatal(http.ListenAndServe(port, router))
}
```

## Step 8: Run the Application

Now, run your application:

```bash
go run main.go
```

You should see output similar to:
```
Connected to SQLite database successfully
Database migration completed
Server is running on http://localhost:8080
```

## Step 9: Test the API

You can test your API using tools like curl, Postman, or any HTTP client.

### Create a Task (POST /api/tasks)
```bash
curl -X POST -H "Content-Type: application/json" -d '{"title": "Learn Go", "description": "Study Go programming language", "status": "pending"}' http://localhost:8080/api/tasks
```

### Get All Tasks (GET /api/tasks)
```bash
curl http://localhost:8080/api/tasks
```

### Get a Specific Task (GET /api/tasks/{id})
```bash
curl http://localhost:8080/api/tasks/1
```

### Update a Task (PUT /api/tasks/{id})
```bash
curl -X PUT -H "Content-Type: application/json" -d '{"title": "Learn Go", "description": "Study Go programming language", "status": "completed"}' http://localhost:8080/api/tasks/1
```

### Delete a Task (DELETE /api/tasks/{id})
```bash
curl -X DELETE http://localhost:8080/api/tasks/1
```

## Step 10: Add a Simple Web Interface (Optional)

For a complete application, you might want to add a simple web interface. Let's create a basic HTML page that interacts with our API.

Create a `static` directory and add an `index.html` file:

```bash
mkdir static
```

Create `static/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Task Manager</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
        }
        h1 {
            color: #333;
        }
        .task {
            border: 1px solid #ddd;
            padding: 15px;
            margin-bottom: 10px;
            border-radius: 5px;
        }
        .task h3 {
            margin-top: 0;
        }
        .task-status {
            display: inline-block;
            padding: 3px 8px;
            border-radius: 3px;
            font-size: 12px;
            color: white;
        }
        .pending { background-color: #f0ad4e; }
        .completed { background-color: #5cb85c; }
        .form-group {
            margin-bottom: 15px;
        }
        label {
            display: block;
            margin-bottom: 5px;
        }
        input, textarea, select {
            width: 100%;
            padding: a5px;
            border: 1px solid #ddd;
            border-radius: 3px;
        }
        button {
            background-color: #5cb85c;
            color: white;
            border: none;
            padding: 8px 15px;
            border-radius: 3px;
            cursor: pointer;
        }
        button.delete {
            background-color: #d9534f;
        }
        button.edit {
            background-color: #5bc0de;
        }
    </style>
</head>
<body>
    <h1>Task Manager</h1>
    
    <div id="task-form">
        <h2>Add New Task</h2>
        <div class="form-group">
            <label for="title">Title:</label>
            <input type="text" id="title" name="title" required>
        </div>
        <div class="form-group">
            <label for="description">Description:</label>
            <textarea id="description" name="description" rows="3"></textarea>
        </div>
        <div class="form-group">
            <label for="status">Status:</label>
            <select id="status" name="status">
                <option value="pending">Pending</option>
                <option value="completed">Completed</option>
            </select>
        </div>
        <button onclick="createTask()">Add Task</button>
    </div>

    <h2>Tasks</h2>
    <div id="tasks-container"></div>

    <script>
        // Fetch all tasks when page loads
        document.addEventListener('DOMContentLoaded', fetchTasks);

        // Fetch all tasks
        function fetchTasks() {
            fetch('/api/tasks')
                .then(response => response.json())
                .then(result => {
                    const tasksContainer = document.getElementById('tasks-container');
                    tasksContainer.innerHTML = '';
                    
                    if (result.data && result.data.length > 0) {
                        result.data.forEach(task => {
                            const taskElement = createTaskElement(task);
                            tasksContainer.appendChild(taskElement);
                        });
                    } else {
                        tasksContainer.innerHTML = '<p>No tasks found. Add a new task!</p>';
                    }
                })
                .catch(error => {
                    console.error('Error fetching tasks:', error);
                });
        }

        // Create task element
        function createTaskElement(task) {
            const taskDiv = document.createElement('div');
            taskDiv.className = 'task';
            taskDiv.dataset.id = task.id;

            const statusClass = task.status === 'completed' ? 'completed' : 'pending';
            
            taskDiv.innerHTML = `
                <h3>${task.title}</h3>
                <p>${task.description}</p>
                <p>Status: <span class="task-status ${statusClass}">${task.status}</span></p>
                <button class="edit" onclick="editTask(${task.id})">Edit</button>
                <button class="delete" onclick="deleteTask(${task.id})">Delete</button>
            `;
            
            return taskDiv;
        }

        // Create a new task
        function createTask() {
            const title = document.getElementById('title').value;
            const description = document.getElementById('description').value;
            const status = document.getElementById('status').value;
            
            if (!title) {
                alert('Title is required!');
                return;
            }
            
            const task = {
                title,
                description,
                status
            };
            
            fetch('/api/tasks', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify(task),
            })
            .then(response => response.json())
            .then(result => {
                // Clear form
                document.getElementById('title').value = '';
                document.getElementById('description').value = '';
                document.getElementById('status').value = 'pending';
                
                // Refresh tasks
                fetchTasks();
            })
            .catch(error => {
                console.error('Error creating task:', error);
            });
        }

        // Edit a task
        function editTask(id) {
            // For simplicity, prompt for new values
            fetch(`/api/tasks/${id}`)
                .then(response => response.json())
                .then(result => {
                    const task = result.data;
                    
                    const newTitle = prompt('Update title:', task.title);
                    const newDescription = prompt('Update description:', task.description);
                    const newStatus = confirm('Mark as completed?') ? 'completed' : 'pending';
                    
                    if (newTitle === null) return; // User canceled
                    
                    const updatedTask = {
                        title: newTitle,
                        description: newDescription,
                        status: newStatus
                    };
                    
                    fetch(`/api/tasks/${id}`, {
                        method: 'PUT',
                        headers: {
                            'Content-Type': 'application/json',
                        },
                        body: JSON.stringify(updatedTask),
                    })
                    .then(response => response.json())
                    .then(result => {
                        fetchTasks();
                    })
                    .catch(error => {
                        console.error('Error updating task:', error);
                    });
                });
        }

        // Delete a task
        function deleteTask(id) {
            if (confirm('Are you sure you want to delete this task?')) {
                fetch(`/api/tasks/${id}`, {
                    method: 'DELETE',
                })
                .then(response => response.json())
                .then(result => {
                    fetchTasks();
                })
                .catch(error => {
                    console.error('Error deleting task:', error);
                });
            }
        }
    </script>
</body>
</html>
```

Now, update your `main.go` to serve static files:

```go
package main

import (
    "fmt"
    "log"
    "net/http"

    "github.com/gorilla/mux"
    "github.com/yourusername/go-sqlite-crud/database"
    "github.com/yourusername/go-sqlite-crud/handlers"
)

func main() {
    // Connect to database
    database.ConnectDB()

    // Initialize router
    router := mux.NewRouter()

    // API Routes
    router.HandleFunc("/api/tasks", handlers.GetAllTasks).Methods("GET")
    router.HandleFunc("/api/tasks/{id}", handlers.GetTask).Methods("GET")
    router.HandleFunc("/api/tasks", handlers.CreateTask).Methods("POST")
    router.HandleFunc("/api/tasks/{id}", handlers.UpdateTask).Methods("PUT")
    router.HandleFunc("/api/tasks/{id}", handlers.DeleteTask).Methods("DELETE")

    // Serve static files
    router.PathPrefix("/").Handler(http.FileServer(http.Dir("./static")))

    // Start server
    port := ":8080"
    fmt.Printf("Server is running on http://localhost%s\n", port)
    log.Fatal(http.ListenAndServe(port, router))
}
```

Now when you run the application and navigate to `http://localhost:8080` in your browser, you'll see a simple task management interface.

## Conclusion

Congratulations! You've built a complete CRUD web application using Go and SQLite. This example demonstrates:

1. Setting up a Go web application with proper structure
2. Using GORM with SQLite for database operations
3. Creating RESTful API endpoints for CRUD operations
4. Basic error handling and status codes
5. A simple web interface to interact with the API

You can extend this application by adding:
- User authentication
- More sophisticated input validation
- Pagination for listing tasks
- Categories or tags for tasks
- Search functionality
- Testing

Happy coding!