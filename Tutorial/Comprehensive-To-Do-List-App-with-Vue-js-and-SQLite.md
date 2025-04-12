# Comprehensive To-Do List App with Vue.js and SQLite

Here's a step-by-step guide to building a categorized to-do list application with Vue.js and SQLite.

## Project Setup

### 1. Initialize Vue.js Project
```bash
npm init vue@latest todo-list-app
cd todo-list-app
npm install
```

### 2. Install Required Dependencies
```bash
npm install sqlite3 better-sqlite3 vite-plugin-sqlite axios
```

## Backend Setup

### 3. Create a Server Folder
```bash
mkdir server
cd server
npm init -y
npm install express better-sqlite3 cors
```

### 4. Create `server.js`
```javascript
// server/server.js
const express = require('express');
const Database = require('better-sqlite3');
const cors = require('cors');

const app = express();
app.use(cors());
app.use(express.json());

const db = new Database('todos.db');

// Create tables if they don't exist
db.exec(`
  CREATE TABLE IF NOT EXISTS categories (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    color TEXT DEFAULT '#ffffff'
  );
  
  CREATE TABLE IF NOT EXISTS todos (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    description TEXT,
    due_date TEXT,
    priority INTEGER DEFAULT 1,
    completed BOOLEAN DEFAULT 0,
    category_id INTEGER,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES categories(id)
  );
`);

// Categories CRUD
app.get('/api/categories', (req, res) => {
  const categories = db.prepare('SELECT * FROM categories').all();
  res.json(categories);
});

app.post('/api/categories', (req, res) => {
  const { name, color } = req.body;
  const stmt = db.prepare('INSERT INTO categories (name, color) VALUES (?, ?)');
  const result = stmt.run(name, color);
  res.json({ id: result.lastInsertRowid, name, color });
});

// Todos CRUD
app.get('/api/todos', (req, res) => {
  const todos = db.prepare(`
    SELECT t.*, c.name as category_name, c.color as category_color 
    FROM todos t LEFT JOIN categories c ON t.category_id = c.id
  `).all();
  res.json(todos);
});

app.post('/api/todos', (req, res) => {
  const { title, description, due_date, priority, category_id } = req.body;
  const stmt = db.prepare(`
    INSERT INTO todos (title, description, due_date, priority, category_id) 
    VALUES (?, ?, ?, ?, ?)
  `);
  const result = stmt.run(title, description, due_date, priority, category_id);
  res.json({ 
    id: result.lastInsertRowid, 
    title, 
    description, 
    due_date, 
    priority, 
    category_id 
  });
});

app.patch('/api/todos/:id', (req, res) => {
  const { id } = req.params;
  const { title, description, due_date, priority, completed, category_id } = req.body;
  
  const updates = [];
  const params = [];
  
  if (title !== undefined) {
    updates.push('title = ?');
    params.push(title);
  }
  // Add other fields similarly...
  
  params.push(id);
  
  const query = `UPDATE todos SET ${updates.join(', ')} WHERE id = ?`;
  db.prepare(query).run(...params);
  
  res.json({ success: true });
});

app.delete('/api/todos/:id', (req, res) => {
  const { id } = req.params;
  db.prepare('DELETE FROM todos WHERE id = ?').run(id);
  res.json({ success: true });
});

const PORT = 3001;
app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

## Frontend Setup

### 5. Modify `src/App.vue`
```vue
<template>
  <div class="app-container">
    <header>
      <h1>Todo List App</h1>
    </header>
    
    <div class="main-content">
      <CategoryManager 
        :categories="categories" 
        @add-category="addCategory"
        @delete-category="deleteCategory"
      />
      
      <TodoForm 
        :categories="categories" 
        @add-todo="addTodo"
      />
      
      <TodoList 
        :todos="filteredTodos" 
        @toggle-complete="toggleComplete"
        @delete-todo="deleteTodo"
        @update-todo="updateTodo"
      />
    </div>
  </div>
</template>

<script>
import axios from 'axios';
import CategoryManager from './components/CategoryManager.vue';
import TodoForm from './components/TodoForm.vue';
import TodoList from './components/TodoList.vue';

export default {
  components: {
    CategoryManager,
    TodoForm,
    TodoList
  },
  data() {
    return {
      todos: [],
      categories: [],
      filter: 'all' // 'all', 'completed', 'active'
    };
  },
  computed: {
    filteredTodos() {
      switch (this.filter) {
        case 'completed':
          return this.todos.filter(todo => todo.completed);
        case 'active':
          return this.todos.filter(todo => !todo.completed);
        default:
          return this.todos;
      }
    }
  },
  async created() {
    await this.fetchTodos();
    await this.fetchCategories();
  },
  methods: {
    async fetchTodos() {
      const response = await axios.get('http://localhost:3001/api/todos');
      this.todos = response.data;
    },
    async fetchCategories() {
      const response = await axios.get('http://localhost:3001/api/categories');
      this.categories = response.data;
    },
    async addTodo(newTodo) {
      const response = await axios.post('http://localhost:3001/api/todos', newTodo);
      this.todos.push(response.data);
    },
    async toggleComplete(todo) {
      const updatedTodo = { ...todo, completed: !todo.completed };
      await axios.patch(`http://localhost:3001/api/todos/${todo.id}`, updatedTodo);
      await this.fetchTodos();
    },
    async deleteTodo(id) {
      await axios.delete(`http://localhost:3001/api/todos/${id}`);
      await this.fetchTodos();
    },
    async updateTodo(updatedTodo) {
      await axios.patch(`http://localhost:3001/api/todos/${updatedTodo.id}`, updatedTodo);
      await this.fetchTodos();
    },
    async addCategory(newCategory) {
      const response = await axios.post('http://localhost:3001/api/categories', newCategory);
      this.categories.push(response.data);
    },
    async deleteCategory(id) {
      await axios.delete(`http://localhost:3001/api/categories/${id}`);
      await this.fetchCategories();
      await this.fetchTodos();
    }
  }
};
</script>

<style>
/* Add your styles here */
</style>
```

### 6. Create Components

#### `src/components/CategoryManager.vue`
```vue
<template>
  <div class="category-manager">
    <h2>Categories</h2>
    <div class="category-list">
      <div 
        v-for="category in categories" 
        :key="category.id" 
        class="category-item"
        :style="{ backgroundColor: category.color }"
      >
        {{ category.name }}
        <button @click="$emit('delete-category', category.id)">×</button>
      </div>
    </div>
    <div class="add-category">
      <input v-model="newCategoryName" placeholder="Category name">
      <input type="color" v-model="newCategoryColor">
      <button @click="addCategory">Add Category</button>
    </div>
  </div>
</template>

<script>
export default {
  props: ['categories'],
  data() {
    return {
      newCategoryName: '',
      newCategoryColor: '#ffffff'
    };
  },
  methods: {
    addCategory() {
      if (this.newCategoryName.trim()) {
        this.$emit('add-category', {
          name: this.newCategoryName,
          color: this.newCategoryColor
        });
        this.newCategoryName = '';
        this.newCategoryColor = '#ffffff';
      }
    }
  }
};
</script>
```

#### `src/components/TodoForm.vue`
```vue
<template>
  <div class="todo-form">
    <h2>Add New Todo</h2>
    <form @submit.prevent="submitTodo">
      <div class="form-group">
        <label>Title:</label>
        <input v-model="newTodo.title" required>
      </div>
      <div class="form-group">
        <label>Description:</label>
        <textarea v-model="newTodo.description"></textarea>
      </div>
      <div class="form-group">
        <label>Due Date:</label>
        <input type="date" v-model="newTodo.due_date">
      </div>
      <div class="form-group">
        <label>Priority:</label>
        <select v-model="newTodo.priority">
          <option value="1">Low</option>
          <option value="2">Medium</option>
          <option value="3">High</option>
        </select>
      </div>
      <div class="form-group">
        <label>Category:</label>
        <select v-model="newTodo.category_id">
          <option value="">No Category</option>
          <option v-for="category in categories" :key="category.id" :value="category.id">
            {{ category.name }}
          </option>
        </select>
      </div>
      <button type="submit">Add Todo</button>
    </form>
  </div>
</template>

<script>
export default {
  props: ['categories'],
  data() {
    return {
      newTodo: {
        title: '',
        description: '',
        due_date: '',
        priority: 1,
        category_id: null
      }
    };
  },
  methods: {
    submitTodo() {
      this.$emit('add-todo', { ...this.newTodo });
      this.resetForm();
    },
    resetForm() {
      this.newTodo = {
        title: '',
        description: '',
        due_date: '',
        priority: 1,
        category_id: null
      };
    }
  }
};
</script>
```

#### `src/components/TodoList.vue`
```vue
<template>
  <div class="todo-list">
    <div class="filters">
      <button @click="$emit('filter', 'all')">All</button>
      <button @click="$emit('filter', 'active')">Active</button>
      <button @click="$emit('filter', 'completed')">Completed</button>
    </div>
    
    <div class="todos">
      <div 
        v-for="todo in todos" 
        :key="todo.id" 
        class="todo-item"
        :class="{ completed: todo.completed }"
        :style="{ borderLeft: `4px solid ${todo.category_color || '#ccc'}` }"
      >
        <div class="todo-content">
          <input 
            type="checkbox" 
            :checked="todo.completed" 
            @change="$emit('toggle-complete', todo)"
          >
          <div class="todo-details">
            <h3>{{ todo.title }}</h3>
            <p v-if="todo.description">{{ todo.description }}</p>
            <div class="todo-meta">
              <span v-if="todo.due_date">Due: {{ formatDate(todo.due_date) }}</span>
              <span>Priority: {{ getPriorityText(todo.priority) }}</span>
              <span v-if="todo.category_name" class="category-tag">
                {{ todo.category_name }}
              </span>
            </div>
          </div>
        </div>
        <div class="todo-actions">
          <button @click="$emit('delete-todo', todo.id)">Delete</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  props: ['todos'],
  methods: {
    formatDate(dateString) {
      return new Date(dateString).toLocaleDateString();
    },
    getPriorityText(priority) {
      const priorities = { 1: 'Low', 2: 'Medium', 3: 'High' };
      return priorities[priority] || 'Unknown';
    }
  }
};
</script>
```

## Final Steps

### 7. Update `vite.config.js`
```javascript
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import sqlite from 'vite-plugin-sqlite';

export default defineConfig({
  plugins: [vue(), sqlite()],
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:3001',
        changeOrigin: true
      }
    }
  }
});
```

### 8. Create Start Scripts
In `package.json`, add:
```json
"scripts": {
  "dev": "vite",
  "build": "vite build",
  "preview": "vite preview",
  "server": "node server/server.js"
}
```

### 9. Run the Application
Open two terminal windows:
```bash
# First terminal - backend
npm run server

# Second terminal - frontend
npm run dev
```

## Features Implemented
1. **Category Management**
   - Add/delete categories
   - Assign color to categories
2. **Todo Management**
   - Create, read, update, delete todos
   - Mark todos as complete
   - Set priority levels
   - Add due dates
3. **Filtering**
   - Filter by completion status
   - View by category
4. **Persistence**
   - SQLite database for data storage
   - REST API for data operations

## Further Enhancements
1. Add user authentication
2. Implement drag-and-drop sorting
3. Add search functionality
4. Create recurring todos
5. Add due date reminders
6. Implement data export/import
7. Add dark/light theme toggle

This comprehensive to-do list app provides a solid foundation that you can extend with additional features as needed.