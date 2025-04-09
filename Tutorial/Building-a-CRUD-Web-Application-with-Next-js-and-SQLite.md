# Building a CRUD Web Application with Next.js and SQLite

I'd be happy to guide you through building a CRUD (Create, Read, Update, Delete) web application with Next.js and SQLite. Let's break this down into manageable steps.

## Step 1: Setting up the project

First, let's create a new Next.js project:

```bash
npx create-next-app@latest crud-nextjs-sqlite
cd crud-nextjs-sqlite
```

Next, let's install the necessary dependencies:

```bash
npm install sqlite sqlite3 better-sqlite3
```

## Step 2: Creating the database structure

Let's create a simple database structure for our application. We'll use a db.js file to handle our database operations:

```javascript
// db.js
import { Database } from 'better-sqlite3';
import path from 'path';
import fs from 'fs';

// Initialize database
const dbDirectory = path.join(process.cwd(), 'db');
const dbPath = path.join(dbDirectory, 'database.sqlite');

// Ensure the db directory exists
if (!fs.existsSync(dbDirectory)) {
  fs.mkdirSync(dbDirectory);
}

// Create or open the database
const db = new Database(dbPath);

// Create tables if they don't exist
const initDb = () => {
  const createTableQuery = `
    CREATE TABLE IF NOT EXISTS items (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      title TEXT NOT NULL,
      description TEXT,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
  `;

  db.exec(createTableQuery);
  console.log('Database initialized');
};

// CRUD operations
// Create
const createItem = (item) => {
  const stmt = db.prepare('INSERT INTO items (title, description) VALUES (?, ?)');
  const result = stmt.run(item.title, item.description);
  return result.lastInsertRowid;
};

// Read all
const getAllItems = () => {
  const stmt = db.prepare('SELECT * FROM items ORDER BY created_at DESC');
  return stmt.all();
};

// Read one
const getItemById = (id) => {
  const stmt = db.prepare('SELECT * FROM items WHERE id = ?');
  return stmt.get(id);
};

// Update
const updateItem = (id, item) => {
  const stmt = db.prepare(`
    UPDATE items 
    SET title = ?, description = ?, updated_at = CURRENT_TIMESTAMP 
    WHERE id = ?
  `);
  const result = stmt.run(item.title, item.description, id);
  return result.changes > 0;
};

// Delete
const deleteItem = (id) => {
  const stmt = db.prepare('DELETE FROM items WHERE id = ?');
  const result = stmt.run(id);
  return result.changes > 0;
};

// Initialize database on import
initDb();

export {
  createItem,
  getAllItems,
  getItemById,
  updateItem,
  deleteItem
};
```

## Step 3: Setting up the API routes

Next, let's create the API routes for our CRUD operations. Create a new folder structure `app/api/items` and add the following files:

### `app/api/items/route.js`

```javascript
// app/api/items/route.js
import { NextResponse } from 'next/server';
import { createItem, getAllItems } from '../../../db';

// GET all items
export async function GET() {
  try {
    const items = getAllItems();
    return NextResponse.json({ items }, { status: 200 });
  } catch (error) {
    console.error('Error fetching items:', error);
    return NextResponse.json({ error: 'Failed to fetch items' }, { status: 500 });
  }
}

// POST create a new item
export async function POST(request) {
  try {
    const body = await request.json();
    
    // Basic validation
    if (!body.title) {
      return NextResponse.json({ error: 'Title is required' }, { status: 400 });
    }
    
    const id = createItem(body);
    return NextResponse.json({ id, success: true }, { status: 201 });
  } catch (error) {
    console.error('Error creating item:', error);
    return NextResponse.json({ error: 'Failed to create item' }, { status: 500 });
  }
}
```

### `app/api/items/[id]/route.js`

```javascript
// app/api/items/[id]/route.js
import { NextResponse } from 'next/server';
import { getItemById, updateItem, deleteItem } from '../../../../db';

// GET a single item by ID
export async function GET(request, { params }) {
  try {
    const id = params.id;
    const item = getItemById(id);
    
    if (!item) {
      return NextResponse.json({ error: 'Item not found' }, { status: 404 });
    }
    
    return NextResponse.json({ item }, { status: 200 });
  } catch (error) {
    console.error('Error fetching item:', error);
    return NextResponse.json({ error: 'Failed to fetch item' }, { status: 500 });
  }
}

// PUT update an item
export async function PUT(request, { params }) {
  try {
    const id = params.id;
    const body = await request.json();
    
    // Basic validation
    if (!body.title) {
      return NextResponse.json({ error: 'Title is required' }, { status: 400 });
    }
    
    const success = updateItem(id, body);
    
    if (!success) {
      return NextResponse.json({ error: 'Item not found' }, { status: 404 });
    }
    
    return NextResponse.json({ success: true }, { status: 200 });
  } catch (error) {
    console.error('Error updating item:', error);
    return NextResponse.json({ error: 'Failed to update item' }, { status: 500 });
  }
}

// DELETE an item
export async function DELETE(request, { params }) {
  try {
    const id = params.id;
    const success = deleteItem(id);
    
    if (!success) {
      return NextResponse.json({ error: 'Item not found' }, { status: 404 });
    }
    
    return NextResponse.json({ success: true }, { status: 200 });
  } catch (error) {
    console.error('Error deleting item:', error);
    return NextResponse.json({ error: 'Failed to delete item' }, { status: 500 });
  }
}
```

## Step 4: Creating the UI components

Now, let's create the UI for our CRUD application:

### `components/ItemForm.js`

```javascript
// components/ItemForm.js
'use client';
import { useState, useEffect } from 'react';
import { useRouter } from 'next/navigation';

export default function ItemForm({ item }) {
  const router = useRouter();
  const [formData, setFormData] = useState({
    title: '',
    description: ''
  });
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [error, setError] = useState('');

  useEffect(() => {
    // If item is provided, we're in edit mode
    if (item) {
      setFormData({
        title: item.title,
        description: item.description || ''
      });
    }
  }, [item]);

  const handleChange = (e) => {
    const { name, value } = e.target;
    setFormData(prev => ({
      ...prev,
      [name]: value
    }));
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setIsSubmitting(true);
    setError('');

    try {
      const url = item 
        ? `/api/items/${item.id}` 
        : '/api/items';
      
      const method = item ? 'PUT' : 'POST';
      
      const response = await fetch(url, {
        method,
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(formData),
      });

      if (!response.ok) {
        const data = await response.json();
        throw new Error(data.error || 'Something went wrong');
      }

      // Redirect to the items list
      router.push('/');
      router.refresh();
    } catch (error) {
      setError(error.message);
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      {error && (
        <div className="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded">
          {error}
        </div>
      )}
      
      <div>
        <label htmlFor="title" className="block text-sm font-medium text-gray-700">
          Title
        </label>
        <input
          type="text"
          id="title"
          name="title"
          value={formData.title}
          onChange={handleChange}
          required
          className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500"
        />
      </div>
      
      <div>
        <label htmlFor="description" className="block text-sm font-medium text-gray-700">
          Description
        </label>
        <textarea
          id="description"
          name="description"
          rows="4"
          value={formData.description}
          onChange={handleChange}
          className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500"
        ></textarea>
      </div>
      
      <div>
        <button
          type="submit"
          disabled={isSubmitting}
          className="w-full flex justify-center py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white bg-blue-600 hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-blue-500"
        >
          {isSubmitting ? 'Saving...' : item ? 'Update Item' : 'Create Item'}
        </button>
      </div>
    </form>
  );
}
```

### `components/ItemList.js`

```javascript
// components/ItemList.js
'use client';
import Link from 'next/link';
import { useRouter } from 'next/navigation';
import { useState } from 'react';

export default function ItemList({ items }) {
  const router = useRouter();
  const [deletingId, setDeletingId] = useState(null);

  const formatDate = (dateString) => {
    return new Date(dateString).toLocaleString();
  };

  const handleDelete = async (id) => {
    if (confirm('Are you sure you want to delete this item?')) {
      setDeletingId(id);
      
      try {
        const response = await fetch(`/api/items/${id}`, {
          method: 'DELETE',
        });

        if (!response.ok) {
          throw new Error('Failed to delete item');
        }

        router.refresh(); // Refresh the page to update the list
      } catch (error) {
        console.error('Error deleting item:', error);
        alert('Failed to delete item');
      } finally {
        setDeletingId(null);
      }
    }
  };

  if (!items || items.length === 0) {
    return (
      <div className="text-center py-10">
        <p className="text-gray-500">No items found. Create your first one!</p>
        <div className="mt-4">
          <Link 
            href="/new" 
            className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700"
          >
            Create Item
          </Link>
        </div>
      </div>
    );
  }

  return (
    <div className="space-y-4">
      <div className="flex justify-between items-center">
        <h2 className="text-xl font-semibold">Items</h2>
        <Link 
          href="/new" 
          className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700"
        >
          Create New
        </Link>
      </div>
      
      <div className="overflow-x-auto">
        <table className="min-w-full divide-y divide-gray-200">
          <thead className="bg-gray-50">
            <tr>
              <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                Title
              </th>
              <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                Description
              </th>
              <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                Created
              </th>
              <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                Updated
              </th>
              <th className="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">
                Actions
              </th>
            </tr>
          </thead>
          <tbody className="bg-white divide-y divide-gray-200">
            {items.map((item) => (
              <tr key={item.id} className="hover:bg-gray-50">
                <td className="px-6 py-4 whitespace-nowrap">
                  <div className="text-sm font-medium text-gray-900">{item.title}</div>
                </td>
                <td className="px-6 py-4">
                  <div className="text-sm text-gray-500 line-clamp-2">{item.description || '-'}</div>
                </td>
                <td className="px-6 py-4 whitespace-nowrap">
                  <div className="text-sm text-gray-500">{formatDate(item.created_at)}</div>
                </td>
                <td className="px-6 py-4 whitespace-nowrap">
                  <div className="text-sm text-gray-500">{formatDate(item.updated_at)}</div>
                </td>
                <td className="px-6 py-4 whitespace-nowrap text-right text-sm font-medium">
                  <Link 
                    href={`/edit/${item.id}`} 
                    className="text-blue-600 hover:text-blue-900 mr-4"
                  >
                    Edit
                  </Link>
                  <button
                    onClick={() => handleDelete(item.id)}
                    disabled={deletingId === item.id}
                    className="text-red-600 hover:text-red-900"
                  >
                    {deletingId === item.id ? 'Deleting...' : 'Delete'}
                  </button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
}
```

## Step 5: Creating the page routes

Let's create the main page routes for our application:

### `app/page.js`

```javascript
// app/page.js
import ItemList from '../components/ItemList';
import { getAllItems } from '../db';

export const dynamic = 'force-dynamic'; // Ensure the page is not cached

export default function Home() {
  const items = getAllItems();
  
  return (
    <main className="container mx-auto px-4 py-8 max-w-6xl">
      <h1 className="text-2xl font-bold mb-6">CRUD App with Next.js and SQLite</h1>
      <ItemList items={items} />
    </main>
  );
}
```

### `app/new/page.js`

```javascript
// app/new/page.js
import Link from 'next/link';
import ItemForm from '../../components/ItemForm';

export default function NewItemPage() {
  return (
    <main className="container mx-auto px-4 py-8 max-w-3xl">
      <div className="mb-6">
        <Link href="/" className="text-blue-600 hover:underline">
          &larr; Back to Items
        </Link>
      </div>
      
      <h1 className="text-2xl font-bold mb-6">Create New Item</h1>
      
      <div className="bg-white shadow rounded-lg p-6">
        <ItemForm />
      </div>
    </main>
  );
}
```

### `app/edit/[id]/page.js`

```javascript
// app/edit/[id]/page.js
import Link from 'next/link';
import { notFound } from 'next/navigation';
import ItemForm from '../../../components/ItemForm';
import { getItemById } from '../../../db';

export const dynamic = 'force-dynamic'; // Disable caching for this page

export default function EditItemPage({ params }) {
  const item = getItemById(params.id);
  
  if (!item) {
    notFound();
  }
  
  return (
    <main className="container mx-auto px-4 py-8 max-w-3xl">
      <div className="mb-6">
        <Link href="/" className="text-blue-600 hover:underline">
          &larr; Back to Items
        </Link>
      </div>
      
      <h1 className="text-2xl font-bold mb-6">Edit Item</h1>
      
      <div className="bg-white shadow rounded-lg p-6">
        <ItemForm item={item} />
      </div>
    </main>
  );
}
```

### `app/not-found.js`

```javascript
// app/not-found.js
import Link from 'next/link';

export default function NotFound() {
  return (
    <div className="min-h-screen flex flex-col items-center justify-center px-4">
      <h1 className="text-4xl font-bold mb-4">404 - Page Not Found</h1>
      <p className="text-gray-600 mb-6">The item you are looking for doesn't exist or has been removed.</p>
      <Link href="/" className="bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700">
        Go Home
      </Link>
    </div>
  );
}
```

## Step 6: Updating the layout

Let's create a nice layout for our application:

### `app/layout.js`
```javascript
// app/layout.js
import './globals.css';
import { Inter } from 'next/font/google';

const inter = Inter({ subsets: ['latin'] });

export const metadata = {
  title: 'CRUD App with Next.js and SQLite',
  description: 'A simple CRUD application built with Next.js and SQLite',
};

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body className={`${inter.className} bg-gray-100 min-h-screen`}>
        <header className="bg-white shadow">
          <div className="container mx-auto px-4 py-6 max-w-6xl">
            <h1 className="text-xl font-semibold text-gray-800">
              Next.js SQLite CRUD App
            </h1>
          </div>
        </header>
        
        {children}
        
        <footer className="bg-white shadow-inner mt-auto">
          <div className="container mx-auto px-4 py-6 max-w-6xl">
            <p className="text-center text-gray-500 text-sm">
              &copy; {new Date().getFullYear()} CRUD App with Next.js and SQLite
            </p>
          </div>
        </footer>
      </body>
    </html>
  );
}
```

## Step 7: Running the application

Now you can run your application:

```bash
npm run dev
```

Your CRUD application will be available at http://localhost:3000.


## What we've built

We've created a complete CRUD application with:

### 1. **Database setup**: 

SQLite database with a "items" table


### 2. **API endpoints**:
* GET /api/items - List all items
* POST /api/items - Create a new item
* GET /api/items/[id] - Get a specific item
* PUT /api/items/[id] - Update an item
* DELETE /api/items/[id] - Delete an item


### 3. **UI Components**:
* ItemList - Displays all items with options to edit or delete
* ItemForm - A reusable form for creating and editing items


### 4. **Pages**:
* Home page (/) - Shows the list of all items
* Create page (/new) - Form to create a new item
* Edit page (/edit/[id]) - Form to edit an existing item


## Additional improvements you could make

### Authentication: 

Add user authentication to secure your application

### Form validation: 

Add more robust form validation

### Pagination: 
Add pagination for the item list

### Search functionality: 
Implement search to filter items

### Categories/Tags: 
Add the ability to categorize items

### Image uploads: 
Allow users to upload images for items

### Rich text editor: 
Add a rich text editor for the description field

### Dark mode: 
Implement a dark mode theme switch