# Next.js CRUD Application with SQLite Authentication

Comprehensive CRUD web application with user authentication using Next.js and SQLite. 

Desgned a full-stack solution with:

- Next.js for the frontend and API routes
- SQLite for database storage
- Authentication system
- Complete CRUD operations

---

## Setting up the project structure and core components

### Project Structure

```
next-auth-crud-app/
├── app/
│   ├── api/
│   │   ├── auth/
│   │   │   └── [...nextauth]/
│   │   │       └── route.js
│   │   ├── items/
│   │   │   ├── route.js
│   │   │   └── [id]/
│   │   │       └── route.js
│   ├── dashboard/
│   │   └── page.js
│   ├── items/
│   │   ├── page.js
│   │   ├── create/
│   │   │   └── page.js
│   │   └── [id]/
│   │       ├── page.js
│   │       └── edit/
│   │           └── page.js
│   ├── login/
│   │   └── page.js
│   ├── register/
│   │   └── page.js
│   ├── layout.js
│   └── page.js
├── components/
│   ├── AuthForm.js
│   ├── ItemForm.js
│   ├── ItemList.js
│   ├── Navbar.js
│   └── ProtectedRoute.js
├── lib/
│   ├── auth.js
│   ├── db.js
│   └── validation.js
├── middleware.js
├── next.config.js
├── package.json
└── README.md
```

### Install the necessary dependencies and set up the initial project: `package.json`

```json
{
  "name": "next-auth-crud-app",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "setup-db": "node lib/setup-db.js"
  },
  "dependencies": {
    "next": "14.0.3",
    "react": "^18",
    "react-dom": "^18",
    "next-auth": "^4.24.5",
    "sqlite": "^5.0.1",
    "sqlite3": "^5.1.6",
    "bcryptjs": "^2.4.3",
    "zod": "^3.22.4",
    "react-hook-form": "^7.48.2",
    "react-hot-toast": "^2.4.1"
  },
  "devDependencies": {
    "autoprefixer": "^10.0.1",
    "postcss": "^8",
    "tailwindcss": "^3.3.0",
    "eslint": "^8",
    "eslint-config-next": "14.0.3"
  }
}
```
### Set up the SQLite database connection and schema: `lib/db.js`

```javascript
import sqlite3 from 'sqlite3';
import { open } from 'sqlite';

// This is a singleton pattern to ensure we only have
// one database connection throughout the app
let db = null;

export async function getDB() {
  if (db) {
    return db;
  }
  
  // Open the database
  db = await open({
    filename: './database.sqlite',
    driver: sqlite3.Database
  });
  
  return db;
}

// Initialize database tables
export async function initDB() {
  const db = await getDB();
  
  // Create users table
  await db.exec(`
    CREATE TABLE IF NOT EXISTS users (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL,
      email TEXT UNIQUE NOT NULL,
      password TEXT NOT NULL,
      createdAt DATETIME DEFAULT CURRENT_TIMESTAMP,
      updatedAt DATETIME DEFAULT CURRENT_TIMESTAMP
    )
  `);
  
  // Create items table
  await db.exec(`
    CREATE TABLE IF NOT EXISTS items (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      title TEXT NOT NULL,
      description TEXT,
      userId INTEGER NOT NULL,
      createdAt DATETIME DEFAULT CURRENT_TIMESTAMP,
      updatedAt DATETIME DEFAULT CURRENT_TIMESTAMP,
      FOREIGN KEY (userId) REFERENCES users(id)
    )
  `);
  
  console.log('Database initialized successfully');
  return db;
}

// Setup script that can be called directly
if (require.main === module) {
  initDB()
    .then(() => {
      console.log('Database setup complete');
      process.exit(0);
    })
    .catch((err) => {
      console.error('Error setting up database:', err);
      process.exit(1);
    });
}
```

### create a setup script to initialize the database: `lib/setup-db.js`

```javascript
// This file is used to set up the database
const sqlite3 = require('sqlite3');
const { open } = require('sqlite');
const bcrypt = require('bcryptjs');

async function setup() {
  // Open the database
  const db = await open({
    filename: './database.sqlite',
    driver: sqlite3.Database
  });
  
  // Create users table
  await db.exec(`
    CREATE TABLE IF NOT EXISTS users (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL,
      email TEXT UNIQUE NOT NULL,
      password TEXT NOT NULL,
      createdAt DATETIME DEFAULT CURRENT_TIMESTAMP,
      updatedAt DATETIME DEFAULT CURRENT_TIMESTAMP
    )
  `);
  
  // Create items table
  await db.exec(`
    CREATE TABLE IF NOT EXISTS items (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      title TEXT NOT NULL,
      description TEXT,
      userId INTEGER NOT NULL,
      createdAt DATETIME DEFAULT CURRENT_TIMESTAMP,
      updatedAt DATETIME DEFAULT CURRENT_TIMESTAMP,
      FOREIGN KEY (userId) REFERENCES users(id)
    )
  `);
  
  // Create test user
  const hashedPassword = await bcrypt.hash('password123', 10);
  try {
    await db.run(
      `INSERT INTO users (name, email, password) VALUES (?, ?, ?)`,
      ['Test User', 'test@example.com', hashedPassword]
    );
    console.log('Test user created');
  } catch (error) {
    // If the user already exists, that's fine
    console.log('Test user may already exist:', error.message);
  }
  
  // Create some test items
  try {
    // Get the user ID first
    const user = await db.get(`SELECT id FROM users WHERE email = ?`, ['test@example.com']);
    
    if (user) {
      // Add some test items
      const items = [
        { title: 'First Item', description: 'This is the first test item', userId: user.id },
        { title: 'Second Item', description: 'This is the second test item', userId: user.id },
        { title: 'Third Item', description: 'This is the third test item', userId: user.id }
      ];
      
      for (const item of items) {
        await db.run(
          `INSERT INTO items (title, description, userId) VALUES (?, ?, ?)`,
          [item.title, item.description, item.userId]
        );
      }
      console.log('Test items created');
    }
  } catch (error) {
    console.log('Error creating test items:', error.message);
  }
  
  console.log('Database setup complete!');
}

setup()
  .then(() => {
    console.log('Done');
    process.exit(0);
  })
  .catch((err) => {
    console.error('Error setting up the database:', err);
    process.exit(1);
  });
```

### set up authentication with NextAuth.js: `app/api/auth/[...nextauth]/route.js`

```javascript
import NextAuth from 'next-auth';
import CredentialsProvider from 'next-auth/providers/credentials';
import { getDB } from '@/lib/db';
import bcrypt from 'bcryptjs';

export const authOptions = {
  providers: [
    CredentialsProvider({
      name: 'Credentials',
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" }
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) {
          return null;
        }

        try {
          const db = await getDB();
          const user = await db.get(
            `SELECT * FROM users WHERE email = ?`, 
            [credentials.email]
          );

          if (!user) {
            return null;
          }

          const isPasswordValid = await bcrypt.compare(
            credentials.password,
            user.password
          );

          if (!isPasswordValid) {
            return null;
          }

          return {
            id: user.id.toString(),
            name: user.name,
            email: user.email
          };
        } catch (error) {
          console.error('Authentication error:', error);
          return null;
        }
      }
    })
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.id = user.id;
      }
      return token;
    },
    async session({ session, token }) {
      if (token) {
        session.user.id = token.id;
      }
      return session;
    }
  },
  pages: {
    signIn: '/login',
  },
  session: {
    strategy: 'jwt',
    maxAge: 30 * 24 * 60 * 60, // 30 days
  },
  secret: process.env.NEXTAUTH_SECRET || 'your-secret-key-change-in-production',
};

const handler = NextAuth(authOptions);

export { handler as GET, handler as POST };
```

### create an authentication utility to handle user registration: `lib/auth.js`

```javascript
import { getDB } from './db';
import bcrypt from 'bcryptjs';
import { z } from 'zod';

// Validation schemas
export const registerSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
});

export const loginSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(1, 'Password is required'),
});

// User registration
export async function registerUser(userData) {
  try {
    // Validate user data
    const validatedData = registerSchema.parse(userData);
    
    const db = await getDB();
    
    // Check if user already exists
    const existingUser = await db.get(
      `SELECT * FROM users WHERE email = ?`, 
      [validatedData.email]
    );
    
    if (existingUser) {
      throw new Error('User with this email already exists');
    }
    
    // Hash the password
    const hashedPassword = await bcrypt.hash(validatedData.password, 10);
    
    // Insert the new user
    const result = await db.run(
      `INSERT INTO users (name, email, password) VALUES (?, ?, ?)`,
      [validatedData.name, validatedData.email, hashedPassword]
    );
    
    if (!result.lastID) {
      throw new Error('Failed to create user');
    }
    
    // Return user without password
    return {
      id: result.lastID,
      name: validatedData.name,
      email: validatedData.email,
    };
  } catch (error) {
    if (error instanceof z.ZodError) {
      throw new Error(error.errors[0].message);
    }
    throw error;
  }
}

// Get user by ID
export async function getUserById(id) {
  try {
    const db = await getDB();
    const user = await db.get(
      `SELECT id, name, email, createdAt, updatedAt FROM users WHERE id = ?`, 
      [id]
    );
    
    if (!user) {
      throw new Error('User not found');
    }
    
    return user;
  } catch (error) {
    throw error;
  }
}

// Get user by email
export async function getUserByEmail(email) {
  try {
    const db = await getDB();
    const user = await db.get(
      `SELECT id, name, email, createdAt, updatedAt FROM users WHERE email = ?`, 
      [email]
    );
    
    if (!user) {
      return null;
    }
    
    return user;
  } catch (error) {
    throw error;
  }
}
```

### create the API routes for CRUD operations on items: `app/api/items/route.js`

```javascript
import { getDB } from '@/lib/db';
import { getServerSession } from 'next-auth/next';
import { authOptions } from '../auth/[...nextauth]/route';
import { NextResponse } from 'next/server';
import { z } from 'zod';

// Validation schema for item creation/update
const itemSchema = z.object({
  title: z.string().min(1, 'Title is required'),
  description: z.string().optional(),
});

// GET handler - fetch all items for the current user
export async function GET(request) {
  try {
    const session = await getServerSession(authOptions);
    
    if (!session) {
      return NextResponse.json(
        { error: 'Unauthorized' },
        { status: 401 }
      );
    }
    
    const userId = session.user.id;
    const db = await getDB();
    
    const items = await db.all(
      `SELECT * FROM items WHERE userId = ? ORDER BY createdAt DESC`,
      [userId]
    );
    
    return NextResponse.json(items);
  } catch (error) {
    console.error('Error fetching items:', error);
    return NextResponse.json(
      { error: 'Failed to fetch items' },
      { status: 500 }
    );
  }
}

// POST handler - create a new item
export async function POST(request) {
  try {
    const session = await getServerSession(authOptions);
    
    if (!session) {
      return NextResponse.json(
        { error: 'Unauthorized' },
        { status: 401 }
      );
    }
    
    const userId = session.user.id;
    const body = await request.json();
    
    // Validate the request body
    try {
      itemSchema.parse(body);
    } catch (error) {
      if (error instanceof z.ZodError) {
        return NextResponse.json(
          { error: error.errors[0].message },
          { status: 400 }
        );
      }
    }
    
    const { title, description = '' } = body;
    
    const db = await getDB();
    const result = await db.run(
      `INSERT INTO items (title, description, userId) VALUES (?, ?, ?)`,
      [title, description, userId]
    );
    
    if (!result.lastID) {
      return NextResponse.json(
        { error: 'Failed to create item' },
        { status: 500 }
      );
    }
    
    const newItem = await db.get(
      `SELECT * FROM items WHERE id = ?`,
      [result.lastID]
    );
    
    return NextResponse.json(newItem, { status: 201 });
  } catch (error) {
    console.error('Error creating item:', error);
    return NextResponse.json(
      { error: 'Failed to create item' },
      { status: 500 }
    );
  }
}
```

### create the API route for individual item operations (update/delete): `app/api/items/[id]/rutes.js`

```javascript
import { getDB } from '@/lib/db';
import { getServerSession } from 'next-auth/next';
import { authOptions } from '../../auth/[...nextauth]/route';
import { NextResponse } from 'next/server';
import { z } from 'zod';

// Validation schema for item update
const updateItemSchema = z.object({
  title: z.string().min(1, 'Title is required'),
  description: z.string().optional(),
});

// GET handler - fetch a single item
export async function GET(request, { params }) {
  try {
    const session = await getServerSession(authOptions);
    
    if (!session) {
      return NextResponse.json(
        { error: 'Unauthorized' },
        { status: 401 }
      );
    }
    
    const userId = session.user.id;
    const { id } = params;
    
    const db = await getDB();
    const item = await db.get(
      `SELECT * FROM items WHERE id = ? AND userId = ?`,
      [id, userId]
    );
    
    if (!item) {
      return NextResponse.json(
        { error: 'Item not found' },
        { status: 404 }
      );
    }
    
    return NextResponse.json(item);
  } catch (error) {
    console.error('Error fetching item:', error);
    return NextResponse.json(
      { error: 'Failed to fetch item' },
      { status: 500 }
    );
  }
}

// PUT handler - update an item
export async function PUT(request, { params }) {
  try {
    const session = await getServerSession(authOptions);
    
    if (!session) {
      return NextResponse.json(
        { error: 'Unauthorized' },
        { status: 401 }
      );
    }
    
    const userId = session.user.id;
    const { id } = params;
    const body = await request.json();
    
    // Validate the request body
    try {
      updateItemSchema.parse(body);
    } catch (error) {
      if (error instanceof z.ZodError) {
        return NextResponse.json(
          { error: error.errors[0].message },
          { status: 400 }
        );
      }
    }
    
    const { title, description = '' } = body;
    
    const db = await getDB();
    
    // Check if the item exists and belongs to the user
    const existingItem = await db.get(
      `SELECT * FROM items WHERE id = ? AND userId = ?`,
      [id, userId]
    );
    
    if (!existingItem) {
      return NextResponse.json(
        { error: 'Item not found or does not belong to you' },
        { status: 404 }
      );
    }
    
    // Update the item
    await db.run(
      `UPDATE items SET title = ?, description = ?, updatedAt = CURRENT_TIMESTAMP WHERE id = ?`,
      [title, description, id]
    );
    
    const updatedItem = await db.get(
      `SELECT * FROM items WHERE id = ?`,
      [id]
    );
    
    return NextResponse.json(updatedItem);
  } catch (error) {
    console.error('Error updating item:', error);
    return NextResponse.json(
      { error: 'Failed to update item' },
      { status: 500 }
    );
  }
}

// DELETE handler - delete an item
export async function DELETE(request, { params }) {
  try {
    const session = await getServerSession(authOptions);
    
    if (!session) {
      return NextResponse.json(
        { error: 'Unauthorized' },
        { status: 401 }
      );
    }
    
    const userId = session.user.id;
    const { id } = params;
    
    const db = await getDB();
    
    // Check if the item exists and belongs to the user
    const existingItem = await db.get(
      `SELECT * FROM items WHERE id = ? AND userId = ?`,
      [id, userId]
    );
    
    if (!existingItem) {
      return NextResponse.json(
        { error: 'Item not found or does not belong to you' },
        { status: 404 }
      );
    }
    
    // Delete the item
    await db.run(
      `DELETE FROM items WHERE id = ?`,
      [id]
    );
    
    return NextResponse.json({ success: true });
  } catch (error) {
    console.error('Error deleting item:', error);
    return NextResponse.json(
      { error: 'Failed to delete item' },
      { status: 500 }
    );
  }
}
```

### create a registration API endpoint: app/api/auth/register/routes.js

```javascript
import { NextResponse } from 'next/server';
import { registerUser } from '@/lib/auth';

export async function POST(request) {
  try {
    const body = await request.json();
    
    const user = await registerUser(body);
    
    return NextResponse.json(
      { user, message: 'User registered successfully' },
      { status: 201 }
    );
  } catch (error) {
    console.error('Registration error:', error);
    
    return NextResponse.json(
      { error: error.message || 'Failed to register user' },
      { status: 400 }
    );
  }
}
```

### create our UI components, starting with the layout and navbar: `app/layout.js`

```javascript
import { Inter } from 'next/font/google';
import './globals.css';
import { Toaster } from 'react-hot-toast';
import Navbar from '@/components/Navbar';
import { getServerSession } from 'next-auth/next';
import { authOptions } from './api/auth/[...nextauth]/route';
import AuthProvider from '@/components/AuthProvider';

const inter = Inter({ subsets: ['latin'] });

export const metadata = {
  title: 'Next.js CRUD with Auth',
  description: 'A comprehensive CRUD application with authentication built with Next.js and SQLite',
};

export default async function RootLayout({ children }) {
  const session = await getServerSession(authOptions);
  
  return (
    <html lang="en">
      <body className={inter.className}>
        <AuthProvider>
          <Navbar session={session} />
          <main className="container mx-auto px-4 py-8">
            {children}
          </main>
          <Toaster position="bottom-right" />
        </AuthProvider>
      </body>
    </html>
  );
}
```

### create the global CSS file: `app/globals.css`

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

:root {
  --foreground-rgb: 0, 0, 0;
  --background-start-rgb: 214, 219, 220;
  --background-end-rgb: 255, 255, 255;
}

@media (prefers-color-scheme: dark) {
  :root {
    --foreground-rgb: 255, 255, 255;
    --background-start-rgb: 0, 0, 0;
    --background-end-rgb: 0, 0, 0;
  }
}

body {
  color: rgb(var(--foreground-rgb));
  background: linear-gradient(
      to bottom,
      transparent,
      rgb(var(--background-end-rgb))
    )
    rgb(var(--background-start-rgb));
}

@layer components {
  .btn {
    @apply px-4 py-2 rounded font-medium transition duration-200;
  }
  
  .btn-primary {
    @apply bg-blue-600 text-white hover:bg-blue-700;
  }
  
  .btn-secondary {
    @apply bg-gray-200 text-gray-800 hover:bg-gray-300;
  }
  
  .btn-danger {
    @apply bg-red-600 text-white hover:bg-red-700;
  }
  
  .form-input {
    @apply w-full p-2 border border-gray-300 rounded focus:outline-none focus:ring-2 focus:ring-blue-500;
  }
  
  .form-label {
    @apply block mb-1 font-medium;
  }
  
  .error-message {
    @apply text-red-600 text-sm mt-1;
  }
  
  .card {
    @apply bg-white rounded-lg shadow-md p-6;
  }
}
```

### create the AuthProvider component: `components/AuthProvider.js`

```javascript
'use client';

import { SessionProvider } from 'next-auth/react';

export default function AuthProvider({ children }) {
  return <SessionProvider>{children}</SessionProvider>;
}
```

### create the Navbar component: `components/Navbar.js`

```javascript
'use client';

import Link from 'next/link';
import { usePathname } from 'next/navigation';
import { signOut, useSession } from 'next-auth/react';

export default function Navbar() {
  const { data: session, status } = useSession();
  const pathname = usePathname();
  const isLoading = status === 'loading';
  
  return (
    <nav className="bg-blue-600 text-white shadow-md">
      <div className="container mx-auto px-4 py-4 flex justify-between items-center">
        <Link href="/" className="text-xl font-bold">
          CRUD App
        </Link>
```

## Run CRUD application with SQLite authentication

To run Next.js CRUD application with SQLite authentication in local development, follow these steps:

### Step 1: Install Dependencies
make sure you have Node.js installed (version 14 or higher is recommended). Then, open a terminal in your project directory and run:

```bash
npm install
```

This will install all the dependencies listed in the package.json file.


### Step 2: Set Up Environment Variables

Create a `.env.local` file in the root of your project with the following content:

```
NEXTAUTH_SECRET=your-secret-key-change-in-production
NEXTAUTH_URL=http://localhost:3000
```

The `NEXTAUTH_SECRET` is used for encrypting the session tokens, and `NEXTAUTH_URL` tells NextAuth.js the base URL of your application.

### Step 3: Initialize the Database

Before starting the application, you need to initialize the SQLite database:

```bash
npm run setup-db
```

This will run the setup script we created earlier that creates the necessary tables and adds some test data.

### Step 4: Start the Development Server

```bash
npm run dev
```

This will start the Next.js development server at http://localhost:3000.

### Step 5: Access the Application

Open your browser and navigate to http://localhost:3000. You should see your CRUD application running.

You can log in with the test account we created during database setup:

- Email: `test@example.com`
- Password: `password123`

Or you can register a new account through the registration page.


### Troubleshooting

If you encounter any issues:

1. ***Database Issues***: If you encounter database-related errors, you can delete the `database.sqlite` file and run `npm run setup-db` again to recreate it.
2. ***NextAuth Issues***: Make sure your `.env.local` file is correctly set up with the environment variables.
3. ***Module Not Found Errors***: If you get "module not found" errors, make sure all dependencies are installed by running `npm install again`.
4. ***Server-Side Rendering Issues***: If you encounter hydration errors, it might be due to mismatches between server and client rendering. Check for components that use browser-specific APIs without proper checks.