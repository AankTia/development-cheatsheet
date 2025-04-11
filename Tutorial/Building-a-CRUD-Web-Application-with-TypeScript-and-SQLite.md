# Building a CRUD Web Application with TypeScript and SQLite

CRUD (Create, Read, Update, Delete) web application using TypeScript and SQLite.

We'll use:

- TypeScript for type safety
- Express.js for the web server
- SQLite for the database
- Basic HTML/CSS for the frontend

---

## Database Layer

Implement a SQLite database connection with proper TypeScript interfaces. The database will:

- Create a table for users if it doesn't exist
- Connect safely with proper error handling
- Support all CRUD operations

---

## Models and Services

The application includes:

- User interface for type safety
- User service with methods for all CRUD operations
- Proper error handling and type checking

---

## Express Routes

The API routes handle:

- GET /api/users - List all users
- GET /api/users/ - Get a single user
- POST /api/users - Create a new user
- PUT /api/users/ - Update a user
- DELETE /api/users/ - Delete a user

---

## Frontend Implementation

The frontend includes:

- Clean HTML/CSS for user management
- JavaScript to interact with the API
- Form for adding/editing users
- Dynamic list of users with edit/delete options

---

## Running the Application

To run the application:

```bash
# Development mode with auto-reload
npm run dev

# Or build and run production
npm run build
npm start
```

The application will be available at http://localhost:3000.

---

### STEP-by-Step Guide to Building Your CRUD Application

#### STEP 1: Project Setup and Dependencies

First, create a new project and install dependencies:

```
/ts-simple-crud
├── src/
│   ├── db/
│   ├── model/
│   ├── routes/
│   ├── public/
│   └── app.ts
├── package.json
└── tsconfig.json
```

1. Initialize Project Directory

   ```bash
   mkdir ts-simple-crud
   cd ts-simple-crud

   npm init -y

   mkdir src src/db src/model src/routes src/public
   ```

2. Install Dependecies

   ```bash
   npm install express sqlite3 cors body-parser
   npm install --save-dev typescript ts-node @types/node @types/express @types/sqlite3 @types/cors nodemon
   ```

### STEP 2: Configure TypeScript (`tsconfig.json`)

Create a tsconfig.json file:

```json
{
  "compilerOptions": {
    "target": "es6",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

### STEP 3: Update package.json with scripts

In package.json, add these scripts:

```json
"scripts": {
  "start": "node dist/app.js",
  "dev": "nodemon --exec ts-node src/app.ts",
  "build": "tsc"
}
```

### STEP 4: Create Database Module (`src/db/database.ts`)

```javascript
import sqlite3 from "sqlite3";
import { open, Database } from "sqlite";

let db: Database;

export async function connectDb() {
  db = await open({
    filename: "./database.sqlite",
    driver: sqlite3.Database,
  });

  await db.exec(`
    CREATE TABLE IF NOT EXISTS users (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL,
      email TEXT UNIQUE NOT NULL,
      created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )
  `);

  console.log("Database connected and initialized");
  return db;
}

export function getDb() {
  if (!db) {
    throw new Error("Database not initialized. Call connectDb first.");
  }
  return db;
}
```

### STEP 5: Create User Model (`src/models/User.ts`)

```javascript
export interface User {
  id?: number;
  name: string;
  email: string;
  created_at?: string;
}

export interface UserInput {
  name: string;
  email: string;
}
```

### STEP 6: Create User Service (`src/models/UserService.ts`)

```javascript
import { getDb } from "../db/database";
import { User, UserInput } from "./User";

export async function getAllUsers(): Promise<User[]> {
  const db = getDb();
  return await db.all("SELECT * FROM users ORDER BY created_at DESC");
}

export async function getUserById(id: number): Promise<User | undefined> {
  const db = getDb();
  return await db.get("SELECT * FROM users WHERE id = ?", id);
}

export async function createUser(user: UserInput): Promise<User> {
  const db = getDb();
  const result = await db.run(
    "INSERT INTO users (name, email) VALUES (?, ?)",
    user.name,
    user.email
  );

  return {
    id: result.lastID,
    ...user,
  };
}

export async function updateUser(
  id: number,
  user: UserInput
): Promise<User | undefined> {
  const db = getDb();
  await db.run(
    "UPDATE users SET name = ?, email = ? WHERE id = ?",
    user.name,
    user.email,
    id
  );

  return await getUserById(id);
}

export async function deleteUser(id: number): Promise<boolean> {
  const db = getDb();
  const result = await db.run("DELETE FROM users WHERE id = ?", id);
  return result.changes > 0;
}
```

### STEP 7: Create API Routes (`src/routes/userRoutes.ts`)

```javascript
import express, { Request, Response } from "express";
import { UserInput } from "../models/User";
import {
  getAllUsers,
  getUserById,
  createUser,
  updateUser,
  deleteUser,
} from "../models/UserService";

const router = express.Router();

// Get all users
router.get("/", async (req: Request, res: Response) => {
  try {
    const users = await getAllUsers();
    res.json(users);
  } catch (error) {
    console.error("Error getting users:", error);
    res.status(500).json({ error: "Failed to retrieve users" });
  }
});

// Get user by ID
router.get("/:id", async (req: Request, res: Response) => {
  try {
    const id = parseInt(req.params.id);
    const user = await getUserById(id);

    if (!user) {
      return res.status(404).json({ error: "User not found" });
    }

    res.json(user);
  } catch (error) {
    console.error("Error getting user:", error);
    res.status(500).json({ error: "Failed to retrieve user" });
  }
});

// Create new user
router.post("/", async (req: Request, res: Response) => {
  try {
    const userInput: UserInput = req.body;

    if (!userInput.name || !userInput.email) {
      return res.status(400).json({ error: "Name and email are required" });
    }

    const newUser = await createUser(userInput);
    res.status(201).json(newUser);
  } catch (error) {
    console.error("Error creating user:", error);

    // Check for unique constraint violation (email already exists)
    if (error.message?.includes("UNIQUE constraint failed")) {
      return res.status(409).json({ error: "Email already exists" });
    }

    res.status(500).json({ error: "Failed to create user" });
  }
});

// Update user
router.put("/:id", async (req: Request, res: Response) => {
  try {
    const id = parseInt(req.params.id);
    const userInput: UserInput = req.body;

    if (!userInput.name || !userInput.email) {
      return res.status(400).json({ error: "Name and email are required" });
    }

    const user = await updateUser(id, userInput);

    if (!user) {
      return res.status(404).json({ error: "User not found" });
    }

    res.json(user);
  } catch (error) {
    console.error("Error updating user:", error);

    if (error.message?.includes("UNIQUE constraint failed")) {
      return res.status(409).json({ error: "Email already exists" });
    }

    res.status(500).json({ error: "Failed to update user" });
  }
});

// Delete user
router.delete("/:id", async (req: Request, res: Response) => {
  try {
    const id = parseInt(req.params.id);
    const success = await deleteUser(id);

    if (!success) {
      return res.status(404).json({ error: "User not found" });
    }

    res.status(204).end();
  } catch (error) {
    console.error("Error deleting user:", error);
    res.status(500).json({ error: "Failed to delete user" });
  }
});

export default router;
```

### STEP 8: Create Main Application (`src/app.ts`)

```javascript
import express from "express";
import cors from "cors";
import bodyParser from "body-parser";
import path from "path";
import { connectDb } from "./db/database";
import userRoutes from "./routes/userRoutes";

const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(cors());
app.use(bodyParser.json());
app.use(express.static(path.join(__dirname, "public")));

// Routes
app.use("/api/users", userRoutes);

// Serve the frontend
app.get("/", (req, res) => {
  res.sendFile(path.join(__dirname, "public", "index.html"));
});

// Start the server
async function startServer() {
  try {
    await connectDb();

    app.listen(PORT, () => {
      console.log(`Server running on port ${PORT}`);
    });
  } catch (error) {
    console.error("Failed to start server:", error);
    process.exit(1);
  }
}

startServer();
```

### STEP 9: Create Frontend

1. `src/public/index.html`

    ```html
    <!DOCTYPE html>
    <html lang="en">
      <head>
        <meta charset="UTF-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1.0" />
        <title>CRUD App with TypeScript and SQLite</title>
        <link rel="stylesheet" href="styles.css" />
      </head>
      <body>
        <div class="container">
          <h1>User Management</h1>

          <div class="form-container">
            <h2 id="form-title">Add New User</h2>
            <form id="user-form">
              <input type="hidden" id="user-id" />
              <div class="form-group">
                <label for="name">Name:</label>
                <input type="text" id="name" required />
              </div>
              <div class="form-group">
                <label for="email">Email:</label>
                <input type="email" id="email" required />
              </div>
              <div class="button-group">
                <button type="submit" id="submit-btn">Add User</button>
                <button type="button" id="cancel-btn" style="display: none;">
                  Cancel
                </button>
              </div>
            </form>
          </div>

          <div class="users-container">
            <h2>Users</h2>
            <div id="users-list"></div>
          </div>
        </div>

        <script src="app.js"></script>
      </body>
    </html>
    ```

2. `src/public/styles.css`

    ```css
    * {
      box-sizing: border-box;
      margin: 0;
      padding: 0;
    }

    body {
      font-family: Arial, sans-serif;
      line-height: 1.6;
      background-color: #f4f4f4;
      color: #333;
    }

    .container {
      max-width: 800px;
      margin: 30px auto;
      padding: 20px;
    }

    h1 {
      text-align: center;
      margin-bottom: 20px;
      color: #2c3e50;
    }

    h2 {
      color: #3498db;
      margin-bottom: 15px;
    }

    .form-container {
      background: #fff;
      padding: 20px;
      border-radius: 5px;
      box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
      margin-bottom: 20px;
    }

    .form-group {
      margin-bottom: 15px;
    }

    label {
      display: block;
      margin-bottom: 5px;
      font-weight: bold;
    }

    input[type="text"],
    input[type="email"] {
      width: 100%;
      padding: 8px;
      border: 1px solid #ccc;
      border-radius: 4px;
    }

    .button-group {
      display: flex;
      gap: 10px;
    }

    button {
      padding: 8px 15px;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      font-weight: bold;
    }

    button[type="submit"] {
      background-color: #2ecc71;
      color: white;
    }

    #cancel-btn {
      background-color: #e74c3c;
      color: white;
    }

    .users-container {
      background: #fff;
      padding: 20px;
      border-radius: 5px;
      box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
    }

    .user-card {
      border: 1px solid #ddd;
      border-radius: 4px;
      padding: 15px;
      margin-bottom: 10px;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }

    .user-info h3 {
      margin-bottom: 5px;
      color: #2c3e50;
    }

    .user-actions {
      display: flex;
      gap: 10px;
    }

    .edit-btn {
      background-color: #3498db;
      color: white;
    }

    .delete-btn {
      background-color: #e74c3c;
      color: white;
    }
    ```

3. `src/public/app.js`

    ```javascript
    document.addEventListener("DOMContentLoaded", () => {
      const userForm = document.getElementById("user-form");
      const formTitle = document.getElementById("form-title");
      const userIdInput = document.getElementById("user-id");
      const nameInput = document.getElementById("name");
      const emailInput = document.getElementById("email");
      const submitBtn = document.getElementById("submit-btn");
      const cancelBtn = document.getElementById("cancel-btn");
      const usersList = document.getElementById("users-list");

      // Load all users on page load
      fetchUsers();

      // Form submission handler
      userForm.addEventListener("submit", async (e) => {
        e.preventDefault();

        const user = {
          name: nameInput.value,
          email: emailInput.value,
        };

        const userId = userIdInput.value;

        if (userId) {
          // Update existing user
          await updateUser(userId, user);
        } else {
          // Create new user
          await createUser(user);
        }

        // Reset form and reload users
        resetForm();
        fetchUsers();
      });

      // Cancel button handler
      cancelBtn.addEventListener("click", resetForm);

      // Fetch all users from the API
      async function fetchUsers() {
        try {
          const response = await fetch("/api/users");
          const users = await response.json();

          displayUsers(users);
        } catch (error) {
          console.error("Error fetching users:", error);
          alert("Failed to load users");
        }
      }

      // Display users in the list
      function displayUsers(users) {
        usersList.innerHTML = "";

        if (users.length === 0) {
          usersList.innerHTML = "<p>No users found</p>";
          return;
        }

        users.forEach((user) => {
          const userCard = document.createElement("div");
          userCard.className = "user-card";

          userCard.innerHTML = `
                    <div class="user-info">
                        <h3>${user.name}</h3>
                        <p>${user.email}</p>
                        <small>Created: ${new Date(
                          user.created_at
                        ).toLocaleString()}</small>
                    </div>
                    <div class="user-actions">
                        <button class="edit-btn" data-id="${user.id}">Edit</button>
                        <button class="delete-btn" data-id="${
                          user.id
                        }">Delete</button>
                    </div>
                `;

          usersList.appendChild(userCard);
        });

        // Add event listeners to edit and delete buttons
        document.querySelectorAll(".edit-btn").forEach((btn) => {
          btn.addEventListener("click", () => editUser(btn.dataset.id));
        });

        document.querySelectorAll(".delete-btn").forEach((btn) => {
          btn.addEventListener("click", () => deleteUser(btn.dataset.id));
        });
      }

      // Create a new user
      async function createUser(user) {
        try {
          const response = await fetch("/api/users", {
            method: "POST",
            headers: {
              "Content-Type": "application/json",
            },
            body: JSON.stringify(user),
          });

          if (!response.ok) {
            const error = await response.json();
            throw new Error(error.error || "Failed to create user");
          }

          alert("User created successfully!");
        } catch (error) {
          console.error("Error creating user:", error);
          alert(error.message);
        }
      }

      // Load user data for editing
      async function editUser(id) {
        try {
          const response = await fetch(`/api/users/${id}`);
          const user = await response.json();

          // Populate the form
          userIdInput.value = user.id;
          nameInput.value = user.name;
          emailInput.value = user.email;

          // Update form UI
          formTitle.textContent = "Edit User";
          submitBtn.textContent = "Update User";
          cancelBtn.style.display = "block";

          // Scroll to form
          document
            .querySelector(".form-container")
            .scrollIntoView({ behavior: "smooth" });
        } catch (error) {
          console.error("Error loading user data:", error);
          alert("Failed to load user data for editing");
        }
      }

      // Update an existing user
      async function updateUser(id, user) {
        try {
          const response = await fetch(`/api/users/${id}`, {
            method: "PUT",
            headers: {
              "Content-Type": "application/json",
            },
            body: JSON.stringify(user),
          });

          if (!response.ok) {
            const error = await response.json();
            throw new Error(error.error || "Failed to update user");
          }

          alert("User updated successfully!");
        } catch (error) {
          console.error("Error updating user:", error);
          alert(error.message);
        }
      }

      // Delete a user
      async function deleteUser(id) {
        if (!confirm("Are you sure you want to delete this user?")) {
          return;
        }

        try {
          const response = await fetch(`/api/users/${id}`, {
            method: "DELETE",
          });

          if (!response.ok) {
            const error = await response.json();
            throw new Error(error.error || "Failed to delete user");
          }

          alert("User deleted successfully!");
          fetchUsers();
        } catch (error) {
          console.error("Error deleting user:", error);
          alert(error.message);
        }
      }

      // Reset the form to its initial state
      function resetForm() {
        userForm.reset();
        userIdInput.value = "";
        formTitle.textContent = "Add New User";
        submitBtn.textContent = "Add User";
        cancelBtn.style.display = "none";
      }
    });
    ```
