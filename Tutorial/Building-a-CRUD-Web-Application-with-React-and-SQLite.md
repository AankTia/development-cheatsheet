# Building a CRUD Web Application with React and SQLite

Here's a step-by-step guide to creating a full-stack CRUD (Create, Read, Update, Delete) application using React for the frontend and SQLite for the database.

## Step 1: Set Up the Project Structure

First, let's create the basic project structure:

1. Create a new project folder
2. Initialize both frontend (React) and backend (Node.js with SQLite) parts

```bash
# Create project directory
mkdir react-sqlite-crud
cd react-sqlite-crud

# Create React frontend
npx create-react-app frontend

# Create Node.js backend
mkdir backend
cd backend
npm init -y
```

## Step 2: Set Up the Backend (Node.js with SQLite)

Install necessary backend dependencies:

```bash
cd backend
npm install express sqlite3 cors body-parser
```

Create `backend/server.js`:

```javascript
const express = require('express');
const sqlite3 = require('sqlite3').verbose();
const bodyParser = require('body-parser');
const cors = require('cors');

const app = express();
const port = 5000;

// Middleware
app.use(cors());
app.use(bodyParser.json());

// Database setup
const db = new sqlite3.Database('./database.db', (err) => {
  if (err) {
    console.error(err.message);
  }
  console.log('Connected to the SQLite database.');
});

// Create items table if it doesn't exist
db.run(`CREATE TABLE IF NOT EXISTS items (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  description TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)`);

// CRUD Routes

// Create
app.post('/api/items', (req, res) => {
  const { name, description } = req.body;
  db.run(
    'INSERT INTO items (name, description) VALUES (?, ?)',
    [name, description],
    function (err) {
      if (err) {
        return res.status(400).json({ error: err.message });
      }
      res.json({ id: this.lastID, name, description });
    }
  );
});

// Read all
app.get('/api/items', (req, res) => {
  db.all('SELECT * FROM items', [], (err, rows) => {
    if (err) {
      return res.status(400).json({ error: err.message });
    }
    res.json(rows);
  });
});

// Read one
app.get('/api/items/:id', (req, res) => {
  const { id } = req.params;
  db.get('SELECT * FROM items WHERE id = ?', [id], (err, row) => {
    if (err) {
      return res.status(400).json({ error: err.message });
    }
    res.json(row);
  });
});

// Update
app.put('/api/items/:id', (req, res) => {
  const { id } = req.params;
  const { name, description } = req.body;
  db.run(
    'UPDATE items SET name = ?, description = ? WHERE id = ?',
    [name, description, id],
    function (err) {
      if (err) {
        return res.status(400).json({ error: err.message });
      }
      res.json({ id, name, description });
    }
  );
});

// Delete
app.delete('/api/items/:id', (req, res) => {
  const { id } = req.params;
  db.run('DELETE FROM items WHERE id = ?', [id], function (err) {
    if (err) {
      return res.status(400).json({ error: err.message });
    }
    res.json({ message: 'Item deleted successfully' });
  });
});

// Start server
app.listen(port, () => {
  console.log(`Server running on http://localhost:${port}`);
});
```

## Step 3: Set Up the React Frontend

Install necessary frontend dependencies:

```bash
cd ../frontend
npm install axios react-router-dom
```

### Create React Components

1. Create `frontend/src/components/ItemList.js`:

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { Link } from 'react-router-dom';

const ItemList = () => {
  const [items, setItems] = useState([]);

  useEffect(() => {
    fetchItems();
  }, []);

  const fetchItems = async () => {
    try {
      const response = await axios.get('http://localhost:5000/api/items');
      setItems(response.data);
    } catch (error) {
      console.error('Error fetching items:', error);
    }
  };

  const handleDelete = async (id) => {
    try {
      await axios.delete(`http://localhost:5000/api/items/${id}`);
      fetchItems();
    } catch (error) {
      console.error('Error deleting item:', error);
    }
  };

  return (
    <div>
      <h2>Items List</h2>
      <Link to="/add" className="btn btn-primary mb-3">
        Add Item
      </Link>
      <table className="table">
        <thead>
          <tr>
            <th>Name</th>
            <th>Description</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {items.map((item) => (
            <tr key={item.id}>
              <td>{item.name}</td>
              <td>{item.description}</td>
              <td>
                <Link
                  to={`/edit/${item.id}`}
                  className="btn btn-sm btn-warning mr-2"
                >
                  Edit
                </Link>
                <button
                  onClick={() => handleDelete(item.id)}
                  className="btn btn-sm btn-danger"
                >
                  Delete
                </button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
};

export default ItemList;
```

2. Create `frontend/src/components/ItemForm.js`:

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { useParams, useNavigate } from 'react-router-dom';

const ItemForm = () => {
  const [item, setItem] = useState({ name: '', description: '' });
  const { id } = useParams();
  const navigate = useNavigate();

  useEffect(() => {
    if (id) {
      fetchItem();
    }
  }, [id]);

  const fetchItem = async () => {
    try {
      const response = await axios.get(`http://localhost:5000/api/items/${id}`);
      setItem(response.data);
    } catch (error) {
      console.error('Error fetching item:', error);
    }
  };

  const handleChange = (e) => {
    setItem({ ...item, [e.target.name]: e.target.value });
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      if (id) {
        await axios.put(`http://localhost:5000/api/items/${id}`, item);
      } else {
        await axios.post('http://localhost:5000/api/items', item);
      }
      navigate('/');
    } catch (error) {
      console.error('Error saving item:', error);
    }
  };

  return (
    <div>
      <h2>{id ? 'Edit Item' : 'Add Item'}</h2>
      <form onSubmit={handleSubmit}>
        <div className="form-group">
          <label>Name</label>
          <input
            type="text"
            name="name"
            className="form-control"
            value={item.name}
            onChange={handleChange}
            required
          />
        </div>
        <div className="form-group">
          <label>Description</label>
          <textarea
            name="description"
            className="form-control"
            value={item.description}
            onChange={handleChange}
          />
        </div>
        <button type="submit" className="btn btn-primary mt-3">
          Save
        </button>
      </form>
    </div>
  );
};

export default ItemForm;
```

3. Update `frontend/src/App.js`:

```javascript
import React from 'react';
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import ItemList from './components/ItemList';
import ItemForm from './components/ItemForm';
import './App.css';

function App() {
  return (
    <Router>
      <div className="container mt-5">
        <Routes>
          <Route path="/" element={<ItemList />} />
          <Route path="/add" element={<ItemForm />} />
          <Route path="/edit/:id" element={<ItemForm />} />
        </Routes>
      </div>
    </Router>
  );
}

export default App;
```

4. Update `frontend/src/index.css` to add some basic styling:

```css
body {
  margin: 0;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Oxygen',
    'Ubuntu', 'Cantarell', 'Fira Sans', 'Droid Sans', 'Helvetica Neue',
    sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.table {
  width: 100%;
  margin-top: 20px;
}

.btn {
  padding: 5px 10px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  text-decoration: none;
  display: inline-block;
}

.btn-primary {
  background-color: #007bff;
  color: white;
}

.btn-warning {
  background-color: #ffc107;
  color: black;
}

.btn-danger {
  background-color: #dc3545;
  color: white;
}

.form-group {
  margin-bottom: 15px;
}

.form-control {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}
```

## Step 4: Run the Application

1. Start the backend server:

```bash
cd backend
node server.js
```

2. In another terminal, start the React frontend:

```bash
cd frontend
npm start
```

## Step 5: Test the Application

1. Open your browser to `http://localhost:3000`
2. You should see an empty items list with an "Add Item" button
3. Test all CRUD operations:
   - Create: Add new items
   - Read: View the list of items
   - Update: Edit existing items
   - Delete: Remove items

## Additional Improvements

For a production-ready application, consider:

1. Adding input validation
2. Implementing error handling and user feedback
3. Adding authentication
4. Using environment variables for configuration
5. Adding tests
6. Implementing pagination for large datasets
7. Using a more sophisticated state management solution (like Redux or Context API) for complex applications

This completes your basic CRUD application with React and SQLite!