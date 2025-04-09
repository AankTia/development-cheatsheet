# Spring Boot SQLite CRUD Application

A simple task management application built with Spring Boot and SQLite, implementing CRUD (Create, Read, Update, Delete) operations.

## Features

- Create, read, update, and delete tasks
- Mark tasks as completed/incomplete
- Search tasks by keyword
- Filter tasks by completion status
- Responsive frontend built with Bootstrap 5
- RESTful API backend
- SQLite database for easy setup with no external dependencies

## Technology Stack

- **Backend**: Spring Boot 2.7.5, Spring Data JPA
- **Database**: SQLite 3
- **Frontend**: HTML, CSS, JavaScript, Bootstrap 5
- **Build Tool**: Maven

## Prerequisites

- Java 11 or higher
- Maven 3.6 or higher

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/yourusername/spring-boot-sqlite-crud.git
cd spring-boot-sqlite-crud
```

### 2. Build the application

```bash
mvn clean install
```

#### Maven project configuration:

```java
// pom.xml

<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" 
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.5</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>task-manager</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>task-manager</name>
    <description>Spring Boot CRUD application with SQLite</description>
    
    <properties>
        <java.version>11</java.version>
    </properties>
    
    <dependencies>
        <!-- Spring Boot Core -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        
        <!-- SQLite Database -->
        <dependency>
            <groupId>org.xerial</groupId>
            <artifactId>sqlite-jdbc</artifactId>
            <version>3.36.0.3</version>
        </dependency>
        
        <!-- Hibernate Dialect for SQLite -->
        <dependency>
            <groupId>com.github.gwenn</groupId>
            <artifactId>sqlite-dialect</artifactId>
            <version>0.1.2</version>
        </dependency>
        
        <!-- Validation -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        
        <!-- Development Tools -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>
        
        <!-- Lombok for reducing boilerplate code -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
    
    <repositories>
        <repository>
            <id>jitpack.io</id>
            <url>https://jitpack.io</url>
        </repository>
    </repositories>
</project>
```

#### Create the main application class (`TaskManagerApplication.java`)

```java
// TaskManagerApplication.java
package com.example.taskmanager;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class TaskManagerApplication {

    public static void main(String[] args) {
        SpringApplication.run(TaskManagerApplication.class, args);
    }
}
```

#### Define the model class for our tasks (`Task.java`)

```java
package com.example.taskmanager.model;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.Table;
import javax.validation.constraints.NotBlank;
import javax.validation.constraints.Size;
import java.time.LocalDateTime;

@Entity
@Table(name = "tasks")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Task {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @NotBlank(message = "Title is required")
    @Size(max = 100, message = "Title must be less than 100 characters")
    private String title;
    
    @Size(max = 500, message = "Description must be less than 500 characters")
    private String description;
    
    private boolean completed = false;
    
    private LocalDateTime createdAt = LocalDateTime.now();
    
    private LocalDateTime updatedAt = LocalDateTime.now();
    
    // Convenient constructor for creating new tasks
    public Task(String title, String description) {
        this.title = title;
        this.description = description;
    }
}
```

#### Create the repository interface to handle database operations: `TaskRepository.java`

```java
package com.example.taskmanager.repository;

import com.example.taskmanager.model.Task;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface TaskRepository extends JpaRepository<Task, Long> {
    
    // Find tasks by completion status
    List<Task> findByCompleted(boolean completed);
    
    // Find tasks containing title (case-insensitive)
    List<Task> findByTitleContainingIgnoreCase(String title);
    
    // Find tasks by title and description (for search functionality)
    List<Task> findByTitleContainingIgnoreCaseOrDescriptionContainingIgnoreCase(String title, String description);
}
```

#### Create the service layer to handle business logic: `TaskService.java`

```java
package com.example.taskmanager.service;

import com.example.taskmanager.model.Task;
import com.example.taskmanager.repository.TaskRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import javax.persistence.EntityNotFoundException;
import java.time.LocalDateTime;
import java.util.List;

@Service
public class TaskService {

    private final TaskRepository taskRepository;

    @Autowired
    public TaskService(TaskRepository taskRepository) {
        this.taskRepository = taskRepository;
    }

    // Create a new task
    public Task createTask(Task task) {
        task.setCreatedAt(LocalDateTime.now());
        task.setUpdatedAt(LocalDateTime.now());
        return taskRepository.save(task);
    }

    // Get all tasks
    public List<Task> getAllTasks() {
        return taskRepository.findAll();
    }

    // Get a single task by ID
    public Task getTaskById(Long id) {
        return taskRepository.findById(id)
                .orElseThrow(() -> new EntityNotFoundException("Task not found with id: " + id));
    }

    // Update a task
    public Task updateTask(Long id, Task taskDetails) {
        Task task = getTaskById(id);
        
        // Update fields
        task.setTitle(taskDetails.getTitle());
        task.setDescription(taskDetails.getDescription());
        task.setCompleted(taskDetails.isCompleted());
        task.setUpdatedAt(LocalDateTime.now());
        
        return taskRepository.save(task);
    }

    // Delete a task
    public void deleteTask(Long id) {
        Task task = getTaskById(id);
        taskRepository.delete(task);
    }
    
    // Mark a task as completed or not completed
    public Task toggleTaskCompletion(Long id) {
        Task task = getTaskById(id);
        task.setCompleted(!task.isCompleted());
        task.setUpdatedAt(LocalDateTime.now());
        return taskRepository.save(task);
    }
    
    // Get tasks by completion status
    public List<Task> getTasksByStatus(boolean completed) {
        return taskRepository.findByCompleted(completed);
    }
    
    // Search tasks by keyword in title or description
    public List<Task> searchTasks(String keyword) {
        return taskRepository.findByTitleContainingIgnoreCaseOrDescriptionContainingIgnoreCase(keyword, keyword);
    }
}
```

#### Create the controller to handle HTTP requests: `TaskController.java`

```java
package com.example.taskmanager.controller;

import com.example.taskmanager.model.Task;
import com.example.taskmanager.service.TaskService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import javax.validation.Valid;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/tasks")
@CrossOrigin(origins = "*") // For development, allow all origins
public class TaskController {

    private final TaskService taskService;

    @Autowired
    public TaskController(TaskService taskService) {
        this.taskService = taskService;
    }

    // Create a new task
    @PostMapping
    public ResponseEntity<Task> createTask(@Valid @RequestBody Task task) {
        Task newTask = taskService.createTask(task);
        return new ResponseEntity<>(newTask, HttpStatus.CREATED);
    }

    // Get all tasks
    @GetMapping
    public ResponseEntity<List<Task>> getAllTasks() {
        List<Task> tasks = taskService.getAllTasks();
        return new ResponseEntity<>(tasks, HttpStatus.OK);
    }

    // Get a single task by ID
    @GetMapping("/{id}")
    public ResponseEntity<Task> getTaskById(@PathVariable Long id) {
        Task task = taskService.getTaskById(id);
        return new ResponseEntity<>(task, HttpStatus.OK);
    }

    // Update a task
    @PutMapping("/{id}")
    public ResponseEntity<Task> updateTask(@PathVariable Long id, @Valid @RequestBody Task taskDetails) {
        Task updatedTask = taskService.updateTask(id, taskDetails);
        return new ResponseEntity<>(updatedTask, HttpStatus.OK);
    }

    // Delete a task
    @DeleteMapping("/{id}")
    public ResponseEntity<Map<String, Boolean>> deleteTask(@PathVariable Long id) {
        taskService.deleteTask(id);
        
        Map<String, Boolean> response = new HashMap<>();
        response.put("deleted", Boolean.TRUE);
        
        return new ResponseEntity<>(response, HttpStatus.OK);
    }
    
    // Toggle task completion status
    @PatchMapping("/{id}/toggle")
    public ResponseEntity<Task> toggleTaskCompletion(@PathVariable Long id) {
        Task updatedTask = taskService.toggleTaskCompletion(id);
        return new ResponseEntity<>(updatedTask, HttpStatus.OK);
    }
    
    // Get tasks by completion status
    @GetMapping("/status")
    public ResponseEntity<List<Task>> getTasksByStatus(@RequestParam boolean completed) {
        List<Task> tasks = taskService.getTasksByStatus(completed);
        return new ResponseEntity<>(tasks, HttpStatus.OK);
    }
    
    // Search tasks
    @GetMapping("/search")
    public ResponseEntity<List<Task>> searchTasks(@RequestParam String keyword) {
        List<Task> tasks = taskService.searchTasks(keyword);
        return new ResponseEntity<>(tasks, HttpStatus.OK);
    }
}
```

#### Set up the application.properties file to configure our SQLite database: `application.roperties`

```
# Server Configuration
server.port=8080

# Database Configuration for SQLite
spring.datasource.url=jdbc:sqlite:taskmanager.db
spring.datasource.driver-class-name=org.sqlite.JDBC
spring.datasource.username=
spring.datasource.password=

# JPA / Hibernate Configuration
spring.jpa.database-platform=org.hibernate.dialect.SQLiteDialect
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# Initialize schema if needed
spring.sql.init.mode=always
spring.sql.init.schema-locations=classpath:schema.sql
spring.sql.init.continue-on-error=true

# Logging Configuration
logging.level.org.springframework=INFO
logging.level.com.example.taskmanager=DEBUG
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

#### Create the schema.sql file to set up our database table: `schema.sql`

```sql
-- Drop table if exists (useful for development)
DROP TABLE IF EXISTS tasks;

-- Create table for tasks
CREATE TABLE IF NOT EXISTS tasks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title VARCHAR(100) NOT NULL,
    description VARCHAR(500),
    completed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- Insert some sample data (will only execute if table is empty)
INSERT OR IGNORE INTO tasks (title, description, completed, created_at, updated_at)
SELECT 'Complete Spring Boot project', 'Implement CRUD operations with SQLite', 0, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP
WHERE NOT EXISTS (SELECT 1 FROM tasks);

INSERT OR IGNORE INTO tasks (title, description, completed, created_at, updated_at)
SELECT 'Learn more about JPA', 'Focus on relationships and advanced queries', 0, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP
WHERE NOT EXISTS (SELECT 1 FROM tasks WHERE title = 'Learn more about JPA');

INSERT OR IGNORE INTO tasks (title, description, completed, created_at, updated_at)
SELECT 'Build a frontend', 'Create a user interface with React or Angular', 0, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP
WHERE NOT EXISTS (SELECT 1 FROM tasks WHERE title = 'Build a frontend');
```

### Create a simple HTML page as a starting point for our frontend: `index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Task Manager</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css" rel="stylesheet">
    <style>
        .completed {
            text-decoration: line-through;
            color: #6c757d;
        }
        .task-card {
            margin-bottom: 15px;
            transition: all 0.3s ease;
        }
        .task-card:hover {
            transform: translateY(-5px);
            box-shadow: 0 10px 20px rgba(0,0,0,0.1);
        }
    </style>
</head>
<body>
    <div class="container mt-5">
        <h1 class="text-center mb-4">Task Manager</h1>
        
        <!-- Add Task Form -->
        <div class="card mb-4">
            <div class="card-header bg-primary text-white">
                <h5 class="mb-0">Add New Task</h5>
            </div>
            <div class="card-body">
                <form id="taskForm">
                    <div class="mb-3">
                        <label for="title" class="form-label">Title</label>
                        <input type="text" class="form-control" id="title" required maxlength="100">
                    </div>
                    <div class="mb-3">
                        <label for="description" class="form-label">Description</label>
                        <textarea class="form-control" id="description" rows="3" maxlength="500"></textarea>
                    </div>
                    <button type="submit" class="btn btn-primary">Save Task</button>
                </form>
            </div>
        </div>
        
        <!-- Search and Filter -->
        <div class="row mb-4">
            <div class="col-md-6">
                <div class="input-group">
                    <input type="text" class="form-control" id="searchInput" placeholder="Search tasks...">
                    <button class="btn btn-outline-secondary" type="button" id="searchButton">Search</button>
                </div>
            </div>
            <div class="col-md-6">
                <select class="form-select" id="statusFilter">
                    <option value="all">All Tasks</option>
                    <option value="active">Active Tasks</option>
                    <option value="completed">Completed Tasks</option>
                </select>
            </div>
        </div>
        
        <!-- Task List -->
        <div id="taskList" class="row">
            <!-- Tasks will be loaded here -->
            <div class="col-12 text-center" id="loadingMessage">
                <div class="spinner-border text-primary" role="status">
                    <span class="visually-hidden">Loading...</span>
                </div>
                <p>Loading tasks...</p>
            </div>
        </div>
    </div>

    <!-- Task Template -->
    <template id="taskTemplate">
        <div class="col-md-6 col-lg-4 task-card">
            <div class="card">
                <div class="card-header d-flex justify-content-between align-items-center">
                    <h5 class="card-title mb-0 task-title">Task Title</h5>
                    <div class="form-check form-switch">
                        <input class="form-check-input task-toggle" type="checkbox">
                    </div>
                </div>
                <div class="card-body">
                    <p class="card-text task-description">Task description goes here.</p>
                    <div class="d-flex justify-content-between align-items-center">
                        <small class="text-muted task-date">Created: Jan 1, 2023</small>
                        <div>
                            <button class="btn btn-sm btn-outline-primary edit-btn">Edit</button>
                            <button class="btn btn-sm btn-outline-danger delete-btn">Delete</button>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </template>

    <!-- Edit Task Modal -->
    <div class="modal fade" id="editTaskModal" tabindex="-1" aria-labelledby="editTaskModalLabel" aria-hidden="true">
        <div class="modal-dialog">
            <div class="modal-content">
                <div class="modal-header">
                    <h5 class="modal-title" id="editTaskModalLabel">Edit Task</h5>
                    <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
                </div>
                <div class="modal-body">
                    <form id="editTaskForm">
                        <input type="hidden" id="editTaskId">
                        <div class="mb-3">
                            <label for="editTitle" class="form-label">Title</label>
                            <input type="text" class="form-control" id="editTitle" required maxlength="100">
                        </div>
                        <div class="mb-3">
                            <label for="editDescription" class="form-label">Description</label>
                            <textarea class="form-control" id="editDescription" rows="3" maxlength="500"></textarea>
                        </div>
                        <div class="form-check mb-3">
                            <input class="form-check-input" type="checkbox" id="editCompleted">
                            <label class="form-check-label" for="editCompleted">
                                Completed
                            </label>
                        </div>
                    </form>
                </div>
                <div class="modal-footer">
                    <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Cancel</button>
                    <button type="button" class="btn btn-primary" id="saveEditBtn">Save Changes</button>
                </div>
            </div>
        </div>
    </div>

    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/js/bootstrap.bundle.min.js"></script>
    <script>
        document.addEventListener('DOMContentLoaded', function() {
            const API_URL = '/api/tasks';
            let editTaskModal;
            
            // Initialize Bootstrap modal
            window.addEventListener('load', function() {
                editTaskModal = new bootstrap.Modal(document.getElementById('editTaskModal'));
            });
            
            // Fetch all tasks
            function fetchTasks() {
                document.getElementById('loadingMessage').style.display = 'block';
                document.getElementById('taskList').innerHTML = '';
                
                fetch(API_URL)
                    .then(response => response.json())
                    .then(tasks => {
                        document.getElementById('loadingMessage').style.display = 'none';
                        if (tasks.length === 0) {
                            document.getElementById('taskList').innerHTML = '<div class="col-12 text-center"><p>No tasks found. Add your first task!</p></div>';
                        } else {
                            renderTasks(tasks);
                        }
                    })
                    .catch(error => {
                        console.error('Error fetching tasks:', error);
                        document.getElementById('loadingMessage').style.display = 'none';
                        document.getElementById('taskList').innerHTML = '<div class="col-12 text-center"><p>Error loading tasks. Please try again.</p></div>';
                    });
            }
            
            // Render tasks
            function renderTasks(tasks) {
                const taskList = document.getElementById('taskList');
                taskList.innerHTML = '';
                const template = document.getElementById('taskTemplate');
                
                tasks.forEach(task => {
                    const taskElement = template.content.cloneNode(true);
                    
                    const title = taskElement.querySelector('.task-title');
                    title.textContent = task.title;
                    if (task.completed) {
                        title.classList.add('completed');
                    }
                    
                    taskElement.querySelector('.task-description').textContent = task.description || 'No description';
                    
                    const createdDate = new Date(task.createdAt);
                    taskElement.querySelector('.task-date').textContent = `Created: ${createdDate.toLocaleDateString()}`;
                    
                    const toggleCheckbox = taskElement.querySelector('.task-toggle');
                    toggleCheckbox.checked = task.completed;
                    toggleCheckbox.addEventListener('change', () => toggleTaskStatus(task.id));
                    
                    const taskCard = taskElement.querySelector('.task-card');
                    taskCard.setAttribute('data-task-id', task.id);
                    
                    const editBtn = taskElement.querySelector('.edit-btn');
                    editBtn.addEventListener('click', () => openEditModal(task));
                    
                    const deleteBtn = taskElement.querySelector('.delete-btn');
                    deleteBtn.addEventListener('click', () => deleteTask(task.id));
                    
                    taskList.appendChild(taskElement);
                });
            }
            
            // Add a new task
            document.getElementById('taskForm').addEventListener('submit', function(e) {
                e.preventDefault();
                
                const title = document.getElementById('title').value.trim();
                const description = document.getElementById('description').value.trim();
                
                if (!title) return;
                
                const newTask = {
                    title: title,
                    description: description,
                    completed: false
                };
                
                fetch(API_URL, {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json'
                    },
                    body: JSON.stringify(newTask)
                })
                .then(response => response.json())
                .then(task => {
                    // Clear form
                    document.getElementById('title').value = '';
                    document.getElementById('description').value = '';
                    
                    // Refresh tasks
                    fetchTasks();
                })
                .catch(error => console.error('Error adding task:', error));
            });
            
            // Toggle task status
            function toggleTaskStatus(taskId) {
                fetch(`${API_URL}/${taskId}/toggle`, {
                    method: 'PATCH'
                })
                .then(response => {
                    if (response.ok) {
                        fetchTasks();
                    }
                })
                .catch(error => console.error('Error toggling task status:', error));
            }
            
            // Open edit modal
            function openEditModal(task) {
                document.getElementById('editTaskId').value = task.id;
                document.getElementById('editTitle').value = task.title;
                document.getElementById('editDescription').value = task.description || '';
                document.getElementById('editCompleted').checked = task.completed;
                
                editTaskModal.show();
            }
            
            // Save edited task
            document.getElementById('saveEditBtn').addEventListener('click', function() {
                const taskId = document.getElementById('editTaskId').value;
                const title = document.getElementById('editTitle').value.trim();
                const description = document.getElementById('editDescription').value.trim();
                const completed = document.getElementById('editCompleted').checked;
                
                if (!title) return;
                
                const updatedTask = {
                    title: title,
                    description: description,
                    completed: completed
                };
                
                fetch(`${API_URL}/${taskId}`, {
                    method: 'PUT',
                    headers: {
                        'Content-Type': 'application/json'
                    },
                    body: JSON.stringify(updatedTask)
                })
                .then(response => {
                    if (response.ok) {
                        editTaskModal.hide();
                        fetchTasks();
                    }
                })
                .catch(error => console.error('Error updating task:', error));
            });

            // Delete a task
            function deleteTask(taskId) {
                if (confirm('Are you sure you want to delete this task?')) {
                    fetch(`${API_URL}/${taskId}`, {
                        method: 'DELETE'
                    })
                    .then(response => {
                        if (response.ok) {
                            fetchTasks();
                        }
                    })
                    .catch(error => console.error('Error deleting task:', error));
                }
            }

            // Search tasks
            document.getElementById('searchButton').addEventListener('click', function() {
                const keyword = document.getElementById('searchInput').value.trim();
                
                if (keyword === '') {
                    fetchTasks();
                    return;
                }
                
                fetch(`${API_URL}/search?keyword=${encodeURIComponent(keyword)}`)
                    .then(response => response.json())
                    .then(tasks => {
                        document.getElementById('statusFilter').value = 'all';
                        renderTasks(tasks);
                    })
                    .catch(error => console.error('Error searching tasks:', error));
            });

            // Filter tasks by status
            document.getElementById('statusFilter').addEventListener('change', function() {
                const statusValue = this.value;
                
                if (statusValue === 'all') {
                    fetchTasks();
                    return;
                }
                
                const completed = statusValue === 'completed';
                
                fetch(`${API_URL}/status?completed=${completed}`)
                    .then(response => response.json())
                    .then(tasks => {
                        document.getElementById('searchInput').value = '';
                        renderTasks(tasks);
                    })
                    .catch(error => console.error('Error filtering tasks:', error));
            });

            // Fetch tasks on page load
            fetchTasks();
        });
    </script>
```

#### Create a test class for our application: `TaskManagerApplicationTest.java`

```java
package com.example.taskmanager;

import com.example.taskmanager.model.Task;
import com.example.taskmanager.repository.TaskRepository;
import com.example.taskmanager.service.TaskService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.boot.web.server.LocalServerPort;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.ActiveProfiles;

import javax.persistence.EntityNotFoundException;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
public class TaskManagerApplicationTests {

    @LocalServerPort
    private int port;

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private TaskRepository taskRepository;

    @Autowired
    private TaskService taskService;

    private String getRootUrl() {
        return "http://localhost:" + port + "/api/tasks";
    }

    @BeforeEach
    public void setup() {
        // Clean up database before each test
        taskRepository.deleteAll();
    }

    @Test
    public void contextLoads() {
        // Basic test to verify the application context loads successfully
        assertThat(taskService).isNotNull();
        assertThat(taskRepository).isNotNull();
    }

    @Test
    public void testCreateTask() {
        Task task = new Task();
        task.setTitle("Test Task");
        task.setDescription("Task for testing");
        task.setCompleted(false);

        ResponseEntity<Task> postResponse = restTemplate.postForEntity(getRootUrl(), task, Task.class);
        assertNotNull(postResponse);
        assertNotNull(postResponse.getBody());
        assertEquals(201, postResponse.getStatusCodeValue());

        // Verify the task was saved correctly
        Task savedTask = postResponse.getBody();
        assertNotNull(savedTask.getId());
        assertEquals("Test Task", savedTask.getTitle());
        assertEquals("Task for testing", savedTask.getDescription());
        assertFalse(savedTask.isCompleted());
    }

    @Test
    public void testGetAllTasks() {
        // Create test tasks
        taskService.createTask(new Task("Task 1", "Description 1"));
        taskService.createTask(new Task("Task 2", "Description 2"));

        ResponseEntity<Task[]> response = restTemplate.getForEntity(getRootUrl(), Task[].class);
        assertNotNull(response.getBody());
        assertEquals(200, response.getStatusCodeValue());
        assertEquals(2, response.getBody().length);
    }

    @Test
    public void testGetTaskById() {
        Task newTask = taskService.createTask(new Task("Get By ID Test", "Test Description"));
        
        ResponseEntity<Task> response = restTemplate.getForEntity(getRootUrl() + "/" + newTask.getId(), Task.class);
        assertNotNull(response.getBody());
        assertEquals(200, response.getStatusCodeValue());
        assertEquals(newTask.getId(), response.getBody().getId());
        assertEquals("Get By ID Test", response.getBody().getTitle());
    }

    @Test
    public void testUpdateTask() {
        Task newTask = taskService.createTask(new Task("Original Title", "Original Description"));

        // Update the task
        newTask.setTitle("Updated Title");
        newTask.setDescription("Updated Description");
        
        HttpHeaders headers = new HttpHeaders();
        HttpEntity<Task> requestEntity = new HttpEntity<>(newTask, headers);

        ResponseEntity<Task> response = restTemplate.exchange(
                getRootUrl() + "/" + newTask.getId(), 
                HttpMethod.PUT, 
                requestEntity, 
                Task.class);

        assertNotNull(response.getBody());
        assertEquals(200, response.getStatusCodeValue());
        assertEquals("Updated Title", response.getBody().getTitle());
        assertEquals("Updated Description", response.getBody().getDescription());
    }

    @Test
    public void testDeleteTask() {
        Task newTask = taskService.createTask(new Task("Task to Delete", "This task will be deleted"));
        
        // First verify the task exists
        assertNotNull(taskRepository.findById(newTask.getId()).orElse(null));
        
        // Delete the task
        Map<String, String> params = new HashMap<>();
        params.put("id", newTask.getId().toString());
        
        restTemplate.delete(getRootUrl() + "/{id}", params);
        
        // Verify the task no longer exists
        assertNull(taskRepository.findById(newTask.getId()).orElse(null));
    }

    @Test
    public void testToggleTaskCompletion() {
        Task newTask = taskService.createTask(new Task("Toggle Task", "This task will be toggled"));
        assertFalse(newTask.isCompleted());
        
        // Toggle task to completed
        ResponseEntity<Task> response = restTemplate.exchange(
                getRootUrl() + "/" + newTask.getId() + "/toggle",
                HttpMethod.PATCH,
                null,
                Task.class);
                
        assertNotNull(response.getBody());
        assertEquals(200, response.getStatusCodeValue());
        assertTrue(response.getBody().isCompleted());
        
        // Toggle back to not completed
        response = restTemplate.exchange(
                getRootUrl() + "/" + newTask.getId() + "/toggle",
                HttpMethod.PATCH,
                null,
                Task.class);
                
        assertNotNull(response.getBody());
        assertFalse(response.getBody().isCompleted());
    }

    @Test
    public void testSearchTasks() {
        taskService.createTask(new Task("Spring Boot", "Learning Spring Boot"));
        taskService.createTask(new Task("JPA Tutorial", "Understanding JPA"));
        taskService.createTask(new Task("REST API", "Building REST API with Spring"));
        
        // Search for tasks with "Spring" in title or description
        ResponseEntity<Task[]> response = restTemplate.getForEntity(
                getRootUrl() + "/search?keyword=Spring",
                Task[].class);
                
        assertNotNull(response.getBody());
        assertEquals(200, response.getStatusCodeValue());
        assertEquals(2, response.getBody().length);  // Should find 2 tasks with "Spring"
    }

    @Test
    public void testGetTasksByStatus() {
        Task task1 = taskService.createTask(new Task("Completed Task", "This is completed"));
        task1.setCompleted(true);
        taskService.updateTask(task1.getId(), task1);
        
        taskService.createTask(new Task("Active Task 1", "This is still active"));
        taskService.createTask(new Task("Active Task 2", "This is also active"));
        
        // Get completed tasks
        ResponseEntity<Task[]> completedResponse = restTemplate.getForEntity(
                getRootUrl() + "/status?completed=true",
                Task[].class);
                
        assertNotNull(completedResponse.getBody());
        assertEquals(1, completedResponse.getBody().length);
        
        // Get active tasks
        ResponseEntity<Task[]> activeResponse = restTemplate.getForEntity(
                getRootUrl() + "/status?completed=false",
                Task[].class);
                
        assertNotNull(activeResponse.getBody());
        assertEquals(2, activeResponse.getBody().length);
    }
}
```

### 3. Run the application

```bash
mvn spring-boot:run
```

The application will start on `http://localhost:8080`

## Project Structure

```
project-structure/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── example/
│   │   │           └── taskmanager/
│   │   │               ├── TaskManagerApplication.java
│   │   │               ├── controller/
│   │   │               │   └── TaskController.java
│   │   │               ├── model/
│   │   │               │   └── Task.java
│   │   │               ├── repository/
│   │   │               │   └── TaskRepository.java
│   │   │               └── service/
│   │   │                   └── TaskService.java
│   │   └── resources/
│   │       ├── application.properties
│   │       ├── schema.sql
│   │       └── static/
│   │           └── index.html
│   └── test/
│       └── java/
│           └── com/
│               └── example/
│                   └── taskmanager/
│                       └── TaskManagerApplicationTests.java
```

## API Endpoints

| Method | URL | Description |
|--------|-----|-------------|
| GET | /api/tasks | Get all tasks |
| GET | /api/tasks/{id} | Get task by ID |
| POST | /api/tasks | Create a new task |
| PUT | /api/tasks/{id} | Update a task |
| DELETE | /api/tasks/{id} | Delete a task |
| PATCH | /api/tasks/{id}/toggle | Toggle task completion status |
| GET | /api/tasks/status?completed={boolean} | Get tasks by completion status |
| GET | /api/tasks/search?keyword={text} | Search tasks by keyword |

## Database Configuration

The application uses an SQLite database located at the root of the project (`taskmanager.db`). The database configuration is set in the `application.properties` file.

## Testing

Run the tests with:

```bash
mvn test
```

## Additional Notes

1. **SQLite Dialect**: The application uses a custom SQLite dialect for Hibernate, provided by the `com.github.gwenn:sqlite-dialect` dependency.

2. **Database Initialization**: The database is initialized with a few sample tasks when the application starts for the first time.

3. **Development Mode**: Spring Boot DevTools is included for enhanced development experience with automatic restarts.

## Extending the Application

### Add User Authentication

1. Add Spring Security dependency to `pom.xml`
2. Create a User entity and repository
3. Implement UserDetailsService
4. Configure SecurityConfig

### Add Task Categories

1. Create a Category entity with a one-to-many relationship to tasks
2. Update the Task entity to include a reference to Category
3. Add Category repository and service
4. Update controllers and frontend to support categories

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the LICENSE file for details.