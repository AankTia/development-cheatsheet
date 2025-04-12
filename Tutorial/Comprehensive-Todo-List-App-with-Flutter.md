# Comprehensive Todo List App with Flutter

Here's a step-by-step guide to building a full-featured todo list application with Flutter. This will include state management, local storage, and various UI components.

## Step 1: Set Up the Project

1. **Create a new Flutter project**:
   ```bash
   flutter create todo_app
   cd todo_app
   ```

2. **Add required dependencies** to `pubspec.yaml`:
   ```yaml
   dependencies:
     flutter:
       sdk: flutter
     provider: ^6.0.5
     hive: ^2.2.3
     hive_flutter: ^1.1.0
     intl: ^0.18.1
     flutter_slidable: ^2.0.0
     datetime_picker_formfield: ^2.0.0
   ```

3. **Run**:
   ```bash
   flutter pub get
   ```

## Step 2: Model Setup

1. **Create a Todo model** in `lib/models/todo.dart`:
   ```dart
   import 'package:hive/hive.dart';

   part 'todo.g.dart';

   @HiveType(typeId: 0)
   class Todo {
     @HiveField(0)
     final String id;
     
     @HiveField(1)
     String title;
     
     @HiveField(2)
     String description;
     
     @HiveField(3)
     bool isCompleted;
     
     @HiveField(4)
     DateTime dueDate;
     
     @HiveField(5)
     String priority; // 'low', 'medium', 'high'

     Todo({
       required this.id,
       required this.title,
       this.description = '',
       this.isCompleted = false,
       required this.dueDate,
       this.priority = 'medium',
     });
   }
   ```

2. **Generate Hive adapters**:
   ```bash
   flutter pub run build_runner build
   ```

## Step 3: Hive Database Setup

1. **Initialize Hive** in `lib/main.dart` before running the app:
   ```dart
   import 'package:hive_flutter/hive_flutter.dart';
   import 'models/todo.dart';

   void main() async {
     WidgetsFlutterBinding.ensureInitialized();
     await Hive.initFlutter();
     Hive.registerAdapter(TodoAdapter());
     await Hive.openBox<Todo>('todos');
     runApp(const MyApp());
   }
   ```

## Step 4: State Management with Provider

1. **Create a TodoProvider** in `lib/providers/todo_provider.dart`:
   ```dart
   import 'package:flutter/material.dart';
   import 'package:hive/hive.dart';
   import '../models/todo.dart';

   class TodoProvider with ChangeNotifier {
     late Box<Todo> _todos;

     TodoProvider() {
       _todos = Hive.box<Todo>('todos');
     }

     List<Todo> get todos => _todos.values.toList();

     List<Todo> get completedTodos => todos.where((todo) => todo.isCompleted).toList();
     
     List<Todo> get pendingTodos => todos.where((todo) => !todo.isCompleted).toList();

     void addTodo(Todo todo) {
       _todos.put(todo.id, todo);
       notifyListeners();
     }

     void updateTodo(Todo todo) {
       _todos.put(todo.id, todo);
       notifyListeners();
     }

     void deleteTodo(String id) {
       _todos.delete(id);
       notifyListeners();
     }

     void toggleTodoStatus(String id) {
       final todo = _todos.get(id);
       if (todo != null) {
         todo.isCompleted = !todo.isCompleted;
         _todos.put(id, todo);
         notifyListeners();
       }
     }
   }
   ```

## Step 5: Main App Structure

1. **Set up Provider** in `lib/main.dart`:
   ```dart
   import 'package:flutter/material.dart';
   import 'package:provider/provider.dart';
   import 'providers/todo_provider.dart';
   import 'screens/home_screen.dart';

   class MyApp extends StatelessWidget {
     const MyApp({super.key});

     @override
     Widget build(BuildContext context) {
       return ChangeNotifierProvider(
         create: (context) => TodoProvider(),
         child: MaterialApp(
           title: 'Todo App',
           theme: ThemeData(
             primarySwatch: Colors.blue,
             visualDensity: VisualDensity.adaptivePlatformDensity,
           ),
           home: const HomeScreen(),
           debugShowCheckedModeBanner: false,
         ),
       );
     }
   }
   ```

## Step 6: Home Screen

1. **Create Home Screen** in `lib/screens/home_screen.dart`:
   ```dart
   import 'package:flutter/material.dart';
   import 'package:flutter_slidable/flutter_slidable.dart';
   import 'package:provider/provider.dart';
   import 'package:todo_app/providers/todo_provider.dart';
   import 'add_edit_screen.dart';
   import '../models/todo.dart';

   class HomeScreen extends StatefulWidget {
     const HomeScreen({super.key});

     @override
     State<HomeScreen> createState() => _HomeScreenState();
   }

   class _HomeScreenState extends State<HomeScreen> {
     int _selectedIndex = 0;

     @override
     Widget build(BuildContext context) {
       return Scaffold(
         appBar: AppBar(
           title: const Text('Todo App'),
           actions: [
             IconButton(
               icon: const Icon(Icons.sort),
               onPressed: _showSortOptions,
             ),
           ],
         ),
         body: _buildTabContent(),
         floatingActionButton: FloatingActionButton(
           onPressed: () => _navigateToAddEditScreen(context),
           child: const Icon(Icons.add),
         ),
         bottomNavigationBar: BottomNavigationBar(
           currentIndex: _selectedIndex,
           onTap: (index) => setState(() => _selectedIndex = index),
           items: const [
             BottomNavigationBarItem(
               icon: Icon(Icons.list),
               label: 'All',
             ),
             BottomNavigationBarItem(
               icon: Icon(Icons.pending_actions),
               label: 'Pending',
             ),
             BottomNavigationBarItem(
               icon: Icon(Icons.done_all),
               label: 'Completed',
             ),
           ],
         ),
       );
     }

     Widget _buildTabContent() {
       final provider = Provider.of<TodoProvider>(context);
       List<Todo> todos = [];

       switch (_selectedIndex) {
         case 0:
           todos = provider.todos;
           break;
         case 1:
           todos = provider.pendingTodos;
           break;
         case 2:
           todos = provider.completedTodos;
           break;
       }

       return todos.isEmpty
           ? const Center(child: Text('No todos found'))
           : ListView.builder(
               itemCount: todos.length,
               itemBuilder: (context, index) {
                 final todo = todos[index];
                 return _buildTodoItem(todo);
               },
             );
     }

     Widget _buildTodoItem(Todo todo) {
       return Card(
         margin: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
         child: Slidable(
           endActionPane: ActionPane(
             motion: const ScrollMotion(),
             children: [
               SlidableAction(
                 onPressed: (context) => _navigateToAddEditScreen(context, todo),
                 backgroundColor: Colors.blue,
                 foregroundColor: Colors.white,
                 icon: Icons.edit,
                 label: 'Edit',
               ),
               SlidableAction(
                 onPressed: (context) => 
                     Provider.of<TodoProvider>(context, listen: false).deleteTodo(todo.id),
                 backgroundColor: Colors.red,
                 foregroundColor: Colors.white,
                 icon: Icons.delete,
                 label: 'Delete',
               ),
             ],
           ),
           child: ListTile(
             leading: Checkbox(
               value: todo.isCompleted,
               onChanged: (value) => 
                   Provider.of<TodoProvider>(context, listen: false).toggleTodoStatus(todo.id),
             ),
             title: Text(
               todo.title,
               style: TextStyle(
                 decoration: todo.isCompleted ? TextDecoration.lineThrough : null,
               ),
             ),
             subtitle: Column(
               crossAxisAlignment: CrossAxisAlignment.start,
               children: [
                 if (todo.description.isNotEmpty) Text(todo.description),
                 Text('Due: ${_formatDate(todo.dueDate)}'),
                 Text('Priority: ${todo.priority}'),
               ],
             ),
             trailing: _getPriorityIcon(todo.priority),
           ),
         ),
       );
     }

     Icon _getPriorityIcon(String priority) {
       switch (priority) {
         case 'high':
           return const Icon(Icons.arrow_upward, color: Colors.red);
         case 'medium':
           return const Icon(Icons.arrow_forward, color: Colors.orange);
         case 'low':
           return const Icon(Icons.arrow_downward, color: Colors.green);
         default:
           return const Icon(Icons.arrow_forward, color: Colors.orange);
       }
     }

     String _formatDate(DateTime date) {
       return '${date.day}/${date.month}/${date.year}';
     }

     void _navigateToAddEditScreen(BuildContext context, [Todo? todo]) {
       Navigator.push(
         context,
         MaterialPageRoute(
           builder: (context) => AddEditScreen(todo: todo),
         ),
       );
     }

     void _showSortOptions() {
       showModalBottomSheet(
         context: context,
         builder: (context) {
           return Column(
             mainAxisSize: MainAxisSize.min,
             children: [
               ListTile(
                 title: const Text('Sort by date (newest first)'),
                 onTap: () {
                   // Implement sorting logic in provider
                   Navigator.pop(context);
                 },
               ),
               ListTile(
                 title: const Text('Sort by date (oldest first)'),
                 onTap: () {
                   // Implement sorting logic in provider
                   Navigator.pop(context);
                 },
               ),
               ListTile(
                 title: const Text('Sort by priority (high to low)'),
                 onTap: () {
                   // Implement sorting logic in provider
                   Navigator.pop(context);
                 },
               ),
               ListTile(
                 title: const Text('Sort by priority (low to high)'),
                 onTap: () {
                   // Implement sorting logic in provider
                   Navigator.pop(context);
                 },
               ),
             ],
           );
         },
       );
     }
   }
   ```

## Step 7: Add/Edit Screen

1. **Create Add/Edit Screen** in `lib/screens/add_edit_screen.dart`:
   ```dart
   import 'package:flutter/material.dart';
   import 'package:provider/provider.dart';
   import 'package:datetime_picker_formfield/datetime_picker_formfield.dart';
   import 'package:intl/intl.dart';
   import '../providers/todo_provider.dart';
   import '../models/todo.dart';

   class AddEditScreen extends StatefulWidget {
     final Todo? todo;

     const AddEditScreen({super.key, this.todo});

     @override
     State<AddEditScreen> createState() => _AddEditScreenState();
   }

   class _AddEditScreenState extends State<AddEditScreen> {
     final _formKey = GlobalKey<FormState>();
     late String _title;
     late String _description;
     late DateTime _dueDate;
     late String _priority;
     final List<String> _priorities = ['low', 'medium', 'high'];

     @override
     void initState() {
       super.initState();
       if (widget.todo != null) {
         _title = widget.todo!.title;
         _description = widget.todo!.description;
         _dueDate = widget.todo!.dueDate;
         _priority = widget.todo!.priority;
       } else {
         _title = '';
         _description = '';
         _dueDate = DateTime.now().add(const Duration(days: 1));
         _priority = 'medium';
       }
     }

     @override
     Widget build(BuildContext context) {
       return Scaffold(
         appBar: AppBar(
           title: Text(widget.todo == null ? 'Add Todo' : 'Edit Todo'),
         ),
         body: Padding(
           padding: const EdgeInsets.all(16.0),
           child: Form(
             key: _formKey,
             child: ListView(
               children: [
                 TextFormField(
                   initialValue: _title,
                   decoration: const InputDecoration(labelText: 'Title'),
                   validator: (value) {
                     if (value == null || value.isEmpty) {
                       return 'Please enter a title';
                     }
                     return null;
                   },
                   onSaved: (value) => _title = value!,
                 ),
                 TextFormField(
                   initialValue: _description,
                   decoration: const InputDecoration(labelText: 'Description'),
                   maxLines: 3,
                   onSaved: (value) => _description = value ?? '',
                 ),
                 const SizedBox(height: 16),
                 DateTimePickerFormField(
                   initialValue: _dueDate,
                   format: DateFormat('yyyy-MM-dd'),
                   decoration: const InputDecoration(
                     labelText: 'Due Date',
                     suffixIcon: Icon(Icons.calendar_today),
                   ),
                   onChanged: (date) => _dueDate = date!,
                   validator: (date) {
                     if (date == null) {
                       return 'Please select a date';
                     }
                     return null;
                   },
                 ),
                 const SizedBox(height: 16),
                 DropdownButtonFormField<String>(
                   value: _priority,
                   items: _priorities.map((priority) {
                     return DropdownMenuItem<String>(
                       value: priority,
                       child: Text(priority[0].toUpperCase() + priority.substring(1)),
                     );
                   }).toList(),
                   decoration: const InputDecoration(labelText: 'Priority'),
                   onChanged: (value) => _priority = value!,
                 ),
                 const SizedBox(height: 24),
                 ElevatedButton(
                   onPressed: _saveTodo,
                   child: const Text('Save'),
                 ),
               ],
             ),
           ),
         ),
       );
     }

     void _saveTodo() {
       if (_formKey.currentState!.validate()) {
         _formKey.currentState!.save();

         final todo = Todo(
           id: widget.todo?.id ?? DateTime.now().toString(),
           title: _title,
           description: _description,
           dueDate: _dueDate,
           priority: _priority,
         );

         final provider = Provider.of<TodoProvider>(context, listen: false);
         if (widget.todo == null) {
           provider.addTodo(todo);
         } else {
           provider.updateTodo(todo);
         }

         Navigator.pop(context);
       }
     }
   }
   ```

## Step 8: Run the App

1. **Run the application**:
   ```bash
   flutter run
   ```

## Additional Features to Consider

1. **Search functionality** - Add a search bar to filter todos
2. **Categories/Tags** - Allow organizing todos by categories
3. **Reminders/Notifications** - Set reminders for important todos
4. **Dark mode support** - Implement theme switching
5. **Backup/Restore** - Add cloud sync or local backup
6. **Statistics** - Show completion statistics
7. **Attachments** - Allow adding images/files to todos

This implementation provides a solid foundation for a todo app with all the essential features. You can extend it further based on your requirements.