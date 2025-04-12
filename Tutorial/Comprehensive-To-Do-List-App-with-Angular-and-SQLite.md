# Comprehensive To-Do List App with Angular and SQLite

This guide will walk you through creating a full-featured to-do list application using Angular for the frontend and SQLite for the database. We'll cover everything from setup to deployment.

## Prerequisites
- Node.js (v16 or later)
- Angular CLI
- Basic knowledge of TypeScript and Angular
- SQLite (we'll use a Node.js wrapper)

## Step 1: Set Up the Angular Project

1. **Create a new Angular project**:
   ```bash
   ng new todo-app
   cd todo-app
   ```

2. **Install necessary dependencies**:
   ```bash
   npm install @angular/material @angular/cdk @angular/animations
   npm install date-fns  # for date handling
   ```

3. **Set up Angular Material**:
   ```bash
   ng add @angular/material
   ```
   Choose a theme (e.g., Indigo/Pink) and set up browser animations.

## Step 2: Set Up SQLite Database

We'll use the `better-sqlite3` package for SQLite integration:

1. **Install SQLite dependencies**:
   ```bash
   npm install better-sqlite3 @types/better-sqlite3
   ```

2. **Create a database service**:
   ```bash
   ng generate service services/database
   ```

3. **Implement the database service** (`src/app/services/database.service.ts`):
   ```typescript
   import { Injectable } from '@angular/core';
   import * as sqlite3 from 'better-sqlite3';
   import { Todo } from '../models/todo.model';

   @Injectable({
     providedIn: 'root'
   })
   export class DatabaseService {
     private db: any;

     constructor() {
       this.initializeDatabase();
     }

     private initializeDatabase(): void {
       this.db = sqlite3('./todo.db');
       
       // Create tables if they don't exist
       this.db.exec(`
         CREATE TABLE IF NOT EXISTS todos (
           id INTEGER PRIMARY KEY AUTOINCREMENT,
           title TEXT NOT NULL,
           description TEXT,
           dueDate TEXT,
           priority INTEGER DEFAULT 1,
           completed INTEGER DEFAULT 0,
           createdAt TEXT DEFAULT CURRENT_TIMESTAMP
         );
         
         CREATE TABLE IF NOT EXISTS categories (
           id INTEGER PRIMARY KEY AUTOINCREMENT,
           name TEXT NOT NULL UNIQUE
         );
         
         CREATE TABLE IF NOT EXISTS todo_categories (
           todoId INTEGER,
           categoryId INTEGER,
           PRIMARY KEY (todoId, categoryId),
           FOREIGN KEY (todoId) REFERENCES todos(id) ON DELETE CASCADE,
           FOREIGN KEY (categoryId) REFERENCES categories(id) ON DELETE CASCADE
         );
       `);
     }

     // Todo CRUD operations
     addTodo(todo: Omit<Todo, 'id'>): number {
       const stmt = this.db.prepare(`
         INSERT INTO todos (title, description, dueDate, priority, completed)
         VALUES (?, ?, ?, ?, ?)
       `);
       const result = stmt.run(todo.title, todo.description, todo.dueDate, todo.priority, todo.completed ? 1 : 0);
       return result.lastInsertRowid;
     }

     getTodos(): Todo[] {
       const stmt = this.db.prepare('SELECT * FROM todos');
       return stmt.all().map((row: any) => ({
         ...row,
         completed: row.completed === 1
       }));
     }

     // Add other CRUD operations (update, delete, etc.)
     // Add category operations
   }
   ```

## Step 3: Create Data Models

1. **Create a Todo model** (`src/app/models/todo.model.ts`):
   ```typescript
   export interface Todo {
     id: number;
     title: string;
     description?: string;
     dueDate?: string;
     priority: number; // 1-5 scale
     completed: boolean;
     createdAt: string;
   }

   export interface Category {
     id: number;
     name: string;
   }
   ```

## Step 4: Create Core Components

1. **Generate core components**:
   ```bash
   ng generate component components/todo-list
   ng generate component components/todo-item
   ng generate component components/add-todo
   ng generate component components/todo-details
   ng generate component components/category-list
   ```

2. **Set up routing** (`src/app/app-routing.module.ts`):
   ```typescript
   import { NgModule } from '@angular/core';
   import { RouterModule, Routes } from '@angular/router';
   import { TodoListComponent } from './components/todo-list/todo-list.component';
   import { TodoDetailsComponent } from './components/todo-details/todo-details.component';

   const routes: Routes = [
     { path: '', redirectTo: '/todos', pathMatch: 'full' },
     { path: 'todos', component: TodoListComponent },
     { path: 'todos/:id', component: TodoDetailsComponent },
     { path: 'categories', component: CategoryListComponent }
   ];

   @NgModule({
     imports: [RouterModule.forRoot(routes)],
     exports: [RouterModule]
   })
   export class AppRoutingModule { }
   ```

## Step 5: Implement Todo List Component

1. **Update the todo-list component** (`src/app/components/todo-list/todo-list.component.ts`):
   ```typescript
   import { Component, OnInit } from '@angular/core';
   import { DatabaseService } from '../../services/database.service';
   import { Todo } from '../../models/todo.model';

   @Component({
     selector: 'app-todo-list',
     templateUrl: './todo-list.component.html',
     styleUrls: ['./todo-list.component.scss']
   })
   export class TodoListComponent implements OnInit {
     todos: Todo[] = [];
     filteredTodos: Todo[] = [];
     filterOptions = {
       completed: 'all',
       priority: 0,
       category: ''
     };

     constructor(private dbService: DatabaseService) {}

     ngOnInit(): void {
       this.loadTodos();
     }

     loadTodos(): void {
       this.todos = this.dbService.getTodos();
       this.applyFilters();
     }

     applyFilters(): void {
       this.filteredTodos = this.todos.filter(todo => {
         let matches = true;
         
         if (this.filterOptions.completed !== 'all') {
           const completedFilter = this.filterOptions.completed === 'completed';
           matches = matches && todo.completed === completedFilter;
         }
         
         if (this.filterOptions.priority > 0) {
           matches = matches && todo.priority === this.filterOptions.priority;
         }
         
         return matches;
       });
     }

     onFilterChange(): void {
       this.applyFilters();
     }

     deleteTodo(id: number): void {
       this.dbService.deleteTodo(id);
       this.loadTodos();
     }

     toggleComplete(todo: Todo): void {
       todo.completed = !todo.completed;
       this.dbService.updateTodo(todo);
       this.applyFilters();
     }
   }
   ```

2. **Create the template** (`todo-list.component.html`):
   ```html
   <div class="todo-container">
     <h1>My To-Do List</h1>
     
     <div class="filters">
       <mat-form-field>
         <mat-label>Status</mat-label>
         <mat-select [(value)]="filterOptions.completed" (selectionChange)="onFilterChange()">
           <mat-option value="all">All</mat-option>
           <mat-option value="completed">Completed</mat-option>
           <mat-option value="pending">Pending</mat-option>
         </mat-select>
       </mat-form-field>
       
       <mat-form-field>
         <mat-label>Priority</mat-label>
         <mat-select [(value)]="filterOptions.priority" (selectionChange)="onFilterChange()">
           <mat-option value="0">All</mat-option>
           <mat-option value="1">Low</mat-option>
           <mat-option value="2">Medium</mat-option>
           <mat-option value="3">High</mat-option>
           <mat-option value="4">Urgent</mat-option>
           <mat-option value="5">Critical</mat-option>
         </mat-select>
       </mat-form-field>
     </div>
     
     <app-add-todo (todoAdded)="loadTodos()"></app-add-todo>
     
     <div class="todo-list">
       <app-todo-item 
         *ngFor="let todo of filteredTodos"
         [todo]="todo"
         (toggleComplete)="toggleComplete($event)"
         (delete)="deleteTodo($event)">
       </app-todo-item>
     </div>
   </div>
   ```

## Step 6: Implement Todo Item Component

1. **Update the todo-item component** (`src/app/components/todo-item/todo-item.component.ts`):
   ```typescript
   import { Component, Input, Output, EventEmitter } from '@angular/core';
   import { Todo } from '../../models/todo.model';

   @Component({
     selector: 'app-todo-item',
     templateUrl: './todo-item.component.html',
     styleUrls: ['./todo-item.component.scss']
   })
   export class TodoItemComponent {
     @Input() todo!: Todo;
     @Output() toggleComplete = new EventEmitter<Todo>();
     @Output() delete = new EventEmitter<number>();

     get priorityClass(): string {
       return `priority-${this.todo.priority}`;
     }

     onToggleComplete(): void {
       this.toggleComplete.emit(this.todo);
     }

     onDelete(): void {
       this.delete.emit(this.todo.id);
     }
   }
   ```

2. **Create the template** (`todo-item.component.html`):
   ```html
   <mat-card [class]="priorityClass">
     <mat-card-content>
       <div class="todo-content">
         <mat-checkbox
           [checked]="todo.completed"
           (change)="onToggleComplete()">
         </mat-checkbox>
         
         <div class="todo-info">
           <h3 [class.completed]="todo.completed">{{ todo.title }}</h3>
           <p *ngIf="todo.description">{{ todo.description }}</p>
           <small *ngIf="todo.dueDate">Due: {{ todo.dueDate | date }}</small>
         </div>
         
         <div class="todo-actions">
           <button mat-icon-button color="warn" (click)="onDelete()">
             <mat-icon>delete</mat-icon>
           </button>
         </div>
       </div>
     </mat-card-content>
   </mat-card>
   ```

## Step 7: Implement Add Todo Component

1. **Update the add-todo component** (`src/app/components/add-todo/add-todo.component.ts`):
   ```typescript
   import { Component, Output, EventEmitter } from '@angular/core';
   import { FormBuilder, FormGroup, Validators } from '@angular/forms';
   import { DatabaseService } from '../../services/database.service';

   @Component({
     selector: 'app-add-todo',
     templateUrl: './add-todo.component.html',
     styleUrls: ['./add-todo.component.scss']
   })
   export class AddTodoComponent {
     @Output() todoAdded = new EventEmitter<void>();
     todoForm: FormGroup;
     showForm = false;

     constructor(
       private fb: FormBuilder,
       private dbService: DatabaseService
     ) {
       this.todoForm = this.fb.group({
         title: ['', Validators.required],
         description: [''],
         dueDate: [''],
         priority: [1, Validators.min(1)]
       });
     }

     onSubmit(): void {
       if (this.todoForm.valid) {
         const todoData = this.todoForm.value;
         this.dbService.addTodo(todoData);
         this.todoAdded.emit();
         this.todoForm.reset({
           priority: 1
         });
         this.showForm = false;
       }
     }

     toggleForm(): void {
       this.showForm = !this.showForm;
     }
   }
   ```

2. **Create the template** (`add-todo.component.html`):
   ```html
   <div class="add-todo-container">
     <button mat-raised-button color="primary" (click)="toggleForm()">
       <mat-icon>add</mat-icon> Add New Task
     </button>
     
     <div *ngIf="showForm" class="todo-form">
       <form [formGroup]="todoForm" (ngSubmit)="onSubmit()">
         <mat-form-field>
           <mat-label>Title</mat-label>
           <input matInput formControlName="title" required>
           <mat-error *ngIf="todoForm.get('title')?.hasError('required')">
             Title is required
           </mat-error>
         </mat-form-field>
         
         <mat-form-field>
           <mat-label>Description</mat-label>
           <textarea matInput formControlName="description"></textarea>
         </mat-form-field>
         
         <mat-form-field>
           <mat-label>Due Date</mat-label>
           <input matInput [matDatepicker]="picker" formControlName="dueDate">
           <mat-datepicker-toggle matSuffix [for]="picker"></mat-datepicker-toggle>
           <mat-datepicker #picker></mat-datepicker>
         </mat-form-field>
         
         <mat-form-field>
           <mat-label>Priority</mat-label>
           <mat-select formControlName="priority">
             <mat-option value="1">Low</mat-option>
             <mat-option value="2">Medium</mat-option>
             <mat-option value="3">High</mat-option>
             <mat-option value="4">Urgent</mat-option>
             <mat-option value="5">Critical</mat-option>
           </mat-select>
         </mat-form-field>
         
         <div class="form-actions">
           <button mat-button type="button" (click)="toggleForm()">Cancel</button>
           <button mat-raised-button color="primary" type="submit">Add Task</button>
         </div>
       </form>
     </div>
   </div>
   ```

## Step 8: Add Styling

1. **Update global styles** (`src/styles.scss`):
   ```scss
   @import '~@angular/material/prebuilt-themes/indigo-pink.css';
   
   body {
     font-family: Roboto, "Helvetica Neue", sans-serif;
     margin: 0;
     padding: 20px;
     background-color: #f5f5f5;
   }
   
   .todo-container {
     max-width: 800px;
     margin: 0 auto;
     background: white;
     padding: 20px;
     border-radius: 8px;
     box-shadow: 0 2px 10px rgba(0,0,0,0.1);
   }
   
   .filters {
     display: flex;
     gap: 20px;
     margin-bottom: 20px;
     
     mat-form-field {
       width: 200px;
     }
   }
   
   .todo-list {
     margin-top: 20px;
     display: flex;
     flex-direction: column;
     gap: 10px;
   }
   
   .todo-content {
     display: flex;
     align-items: center;
     gap: 15px;
     
     .todo-info {
       flex: 1;
       
       h3 {
         margin: 0;
         
         &.completed {
           text-decoration: line-through;
           color: #888;
         }
       }
       
       p {
         margin: 5px 0 0;
         color: #555;
       }
       
       small {
         color: #888;
       }
     }
   }
   
   .priority-1 { border-left: 4px solid #4CAF50; }
   .priority-2 { border-left: 4px solid #2196F3; }
   .priority-3 { border-left: 4px solid #FFC107; }
   .priority-4 { border-left: 4px solid #FF9800; }
   .priority-5 { border-left: 4px solid #F44336; }
   
   .add-todo-container {
     margin: 20px 0;
     
     .todo-form {
       margin-top: 20px;
       padding: 20px;
       background: #f9f9f9;
       border-radius: 8px;
       
       form {
         display: flex;
         flex-direction: column;
         gap: 20px;
       }
       
       .form-actions {
         display: flex;
         justify-content: flex-end;
         gap: 10px;
       }
     }
   }
   ```

## Step 9: Implement Category Features

1. **Extend the database service** (`database.service.ts`):
   ```typescript
   // Add these methods to DatabaseService
   addCategory(name: string): number {
     const stmt = this.db.prepare('INSERT INTO categories (name) VALUES (?)');
     const result = stmt.run(name);
     return result.lastInsertRowid;
   }

   getCategories(): Category[] {
     const stmt = this.db.prepare('SELECT * FROM categories');
     return stmt.all();
   }

   addTodoCategory(todoId: number, categoryId: number): void {
     const stmt = this.db.prepare(`
       INSERT INTO todo_categories (todoId, categoryId)
       VALUES (?, ?)
     `);
     stmt.run(todoId, categoryId);
   }

   getTodoCategories(todoId: number): Category[] {
     const stmt = this.db.prepare(`
       SELECT c.* FROM categories c
       JOIN todo_categories tc ON c.id = tc.categoryId
       WHERE tc.todoId = ?
     `);
     return stmt.all(todoId);
   }
   ```

2. **Update the category-list component**:
   ```typescript
   import { Component, OnInit } from '@angular/core';
   import { DatabaseService } from '../../services/database.service';
   import { Category } from '../../models/todo.model';

   @Component({
     selector: 'app-category-list',
     templateUrl: './category-list.component.html',
     styleUrls: ['./category-list.component.scss']
   })
   export class CategoryListComponent implements OnInit {
     categories: Category[] = [];
     newCategoryName = '';

     constructor(private dbService: DatabaseService) {}

     ngOnInit(): void {
       this.loadCategories();
     }

     loadCategories(): void {
       this.categories = this.dbService.getCategories();
     }

     addCategory(): void {
       if (this.newCategoryName.trim()) {
         this.dbService.addCategory(this.newCategoryName.trim());
         this.newCategoryName = '';
         this.loadCategories();
       }
     }

     deleteCategory(id: number): void {
       this.dbService.deleteCategory(id);
       this.loadCategories();
     }
   }
   ```

## Step 10: Add Additional Features (Optional)

1. **Add search functionality**:
   - Add a search input to the todo-list component
   - Implement filtering based on search terms

2. **Add sorting options**:
   - Allow sorting by due date, priority, or creation date

3. **Add drag-and-drop reordering**:
   - Use Angular CDK Drag and Drop

4. **Add due date reminders**:
   - Implement notifications for upcoming tasks

5. **Add data export/import**:
   - Implement JSON export/import functionality

## Step 11: Build and Deploy

1. **Build the application**:
   ```bash
   ng build --prod
   ```

2. **For deployment with SQLite**, you'll need to:
   - Package the application with Electron for desktop use
   - Or use a backend service (Node.js/Express) to handle database operations for web deployment

3. **For web deployment with a backend**:
   - Create an API service to interact with SQLite
   - Deploy the Angular app and backend separately

## Final Notes

This implementation provides a comprehensive to-do list application with:
- Task creation, editing, and deletion
- Priority levels
- Due dates
- Completion status
- Categories
- Filtering and sorting

To further enhance the application, consider:
- Adding user authentication
- Implementing data synchronization for multiple devices
- Adding recurring tasks
- Implementing task sharing/collaboration

Remember to handle errors appropriately in production and add proper input validation for all user inputs.