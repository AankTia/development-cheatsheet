# Comprehensive To-Do List App for Android

This guide will walk you through creating a full-featured to-do list application for Android using Kotlin and modern Android development practices.

## Prerequisites
- Android Studio (latest version)
- Basic knowledge of Kotlin
- Understanding of Android fundamentals

## Step 1: Set Up the Project

1. **Create a new Android Studio project**
   - Select "Empty Activity" template
   - Name: "TodoListApp"
   - Package name: com.yourname.todolist
   - Language: Kotlin
   - Minimum SDK: API 21 (Android 5.0)

2. **Configure project structure**
   - Enable ViewBinding in build.gradle (Module: app):
     ```gradle
     android {
         buildFeatures {
             viewBinding true
         }
     }
     ```

## Step 2: Design the Data Model

1. **Create a Todo data class**
   ```kotlin
   data class Todo(
       val id: Int,
       val title: String,
       val description: String? = null,
       val dueDate: Long? = null, // timestamp
       val priority: Int = 0, // 0=low, 1=medium, 2=high
       val isCompleted: Boolean = false,
       val createdAt: Long = System.currentTimeMillis()
   )
   ```

2. **Set up Room Database**
   - Add dependencies to build.gradle (Module: app):
     ```gradle
     implementation "androidx.room:room-runtime:2.6.1"
     implementation "androidx.room:room-ktx:2.6.1"
     kapt "androidx.room:room-compiler:2.6.1"
     ```

   - Create Todo entity:
     ```kotlin
     @Entity(tableName = "todos")
     data class Todo(
         @PrimaryKey(autoGenerate = true) val id: Int = 0,
         @ColumnInfo(name = "title") val title: String,
         @ColumnInfo(name = "description") val description: String? = null,
         @ColumnInfo(name = "due_date") val dueDate: Long? = null,
         @ColumnInfo(name = "priority") val priority: Int = 0,
         @ColumnInfo(name = "is_completed") val isCompleted: Boolean = false,
         @ColumnInfo(name = "created_at") val createdAt: Long = System.currentTimeMillis()
     )
     ```

   - Create TodoDao:
     ```kotlin
     @Dao
     interface TodoDao {
         @Insert
         suspend fun insert(todo: Todo)
     
         @Update
         suspend fun update(todo: Todo)
     
         @Delete
         suspend fun delete(todo: Todo)
     
         @Query("SELECT * FROM todos ORDER BY is_completed ASC, priority DESC, due_date ASC")
         fun getAllTodos(): Flow<List<Todo>>
     
         @Query("SELECT * FROM todos WHERE id = :id")
         suspend fun getTodoById(id: Int): Todo?
     }
     ```

   - Create AppDatabase:
     ```kotlin
     @Database(entities = [Todo::class], version = 1, exportSchema = false)
     abstract class AppDatabase : RoomDatabase() {
         abstract fun todoDao(): TodoDao
     
         companion object {
             @Volatile
             private var INSTANCE: AppDatabase? = null
     
             fun getDatabase(context: Context): AppDatabase {
                 return INSTANCE ?: synchronized(this) {
                     val instance = Room.databaseBuilder(
                         context.applicationContext,
                         AppDatabase::class.java,
                         "todo_database"
                     ).build()
                     INSTANCE = instance
                     instance
                 }
             }
         }
     }
     ```

## Step 3: Create Repository

```kotlin
class TodoRepository(private val todoDao: TodoDao) {
    val allTodos: Flow<List<Todo>> = todoDao.getAllTodos()

    suspend fun insert(todo: Todo) {
        todoDao.insert(todo)
    }

    suspend fun update(todo: Todo) {
        todoDao.update(todo)
    }

    suspend fun delete(todo: Todo) {
        todoDao.delete(todo)
    }

    suspend fun getTodoById(id: Int): Todo? {
        return todoDao.getTodoById(id)
    }
}
```

## Step 4: Set Up ViewModel

```kotlin
class TodoViewModel(application: Application) : AndroidViewModel(application) {
    private val repository: TodoRepository
    val allTodos: LiveData<List<Todo>>

    init {
        val todoDao = AppDatabase.getDatabase(application).todoDao()
        repository = TodoRepository(todoDao)
        allTodos = repository.allTodos.asLiveData()
    }

    fun insert(todo: Todo) = viewModelScope.launch {
        repository.insert(todo)
    }

    fun update(todo: Todo) = viewModelScope.launch {
        repository.update(todo)
    }

    fun delete(todo: Todo) = viewModelScope.launch {
        repository.delete(todo)
    }

    fun getTodoById(id: Int): LiveData<Todo?> {
        return liveData {
            val todo = repository.getTodoById(id)
            emit(todo)
        }
    }
}
```

## Step 5: Create UI Components

1. **Todo Item Layout (item_todo.xml)**
   ```xml
   <?xml version="1.0" encoding="utf-8"?>
   <androidx.constraintlayout.widget.ConstraintLayout 
       xmlns:android="http://schemas.android.com/apk/res/android"
       xmlns:app="http://schemas.android.com/apk/res-auto"
       android:layout_width="match_parent"
       android:layout_height="wrap_content"
       android:padding="16dp">
       
       <CheckBox
           android:id="@+id/checkBoxCompleted"
           android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           app:layout_constraintStart_toStartOf="parent"
           app:layout_constraintTop_toTopOf="parent"/>
       
       <TextView
           android:id="@+id/textViewTitle"
           android:layout_width="0dp"
           android:layout_height="wrap_content"
           android:layout_marginStart="16dp"
           android:layout_marginEnd="16dp"
           android:textSize="18sp"
           android:textStyle="bold"
           app:layout_constraintEnd_toStartOf="@id/imageViewPriority"
           app:layout_constraintStart_toEndOf="@id/checkBoxCompleted"
           app:layout_constraintTop_toTopOf="parent"/>
       
       <TextView
           android:id="@+id/textViewDescription"
           android:layout_width="0dp"
           android:layout_height="wrap_content"
           android:layout_marginTop="4dp"
           android:layout_marginStart="16dp"
           android:layout_marginEnd="16dp"
           android:maxLines="2"
           android:ellipsize="end"
           app:layout_constraintEnd_toStartOf="@id/imageViewPriority"
           app:layout_constraintStart_toEndOf="@id/checkBoxCompleted"
           app:layout_constraintTop_toBottomOf="@id/textViewTitle"/>
       
       <TextView
           android:id="@+id/textViewDueDate"
           android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:layout_marginTop="8dp"
           android:textSize="12sp"
           app:layout_constraintStart_toEndOf="@id/checkBoxCompleted"
           app:layout_constraintTop_toBottomOf="@id/textViewDescription"/>
       
       <ImageView
           android:id="@+id/imageViewPriority"
           android:layout_width="24dp"
           android:layout_height="24dp"
           android:layout_marginEnd="8dp"
           app:layout_constraintBottom_toBottomOf="parent"
           app:layout_constraintEnd_toEndOf="parent"
           app:layout_constraintTop_toTopOf="parent"/>
       
   </androidx.constraintlayout.widget.ConstraintLayout>
   ```

2. **Main Activity Layout (activity_main.xml)**
   ```xml
   <?xml version="1.0" encoding="utf-8"?>
   <androidx.coordinatorlayout.widget.CoordinatorLayout 
       xmlns:android="http://schemas.android.com/apk/res/android"
       xmlns:app="http://schemas.android.com/apk/res-auto"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
       
       <androidx.recyclerview.widget.RecyclerView
           android:id="@+id/recyclerViewTodos"
           android:layout_width="match_parent"
           android:layout_height="match_parent"
           android:clipToPadding="false"
           android:paddingBottom="72dp"/>
       
       <com.google.android.material.floatingactionbutton.FloatingActionButton
           android:id="@+id/fabAdd"
           android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:layout_gravity="bottom|end"
           android:layout_margin="16dp"
           android:src="@drawable/ic_add"
           app:backgroundTint="@color/purple_500"
           app:tint="@android:color/white"/>
       
   </androidx.coordinatorlayout.widget.CoordinatorLayout>
   ```

3. **Add/Edit Todo Dialog (dialog_add_edit_todo.xml)**
   ```xml
   <?xml version="1.0" encoding="utf-8"?>
   <ScrollView 
       xmlns:android="http://schemas.android.com/apk/res/android"
       android:layout_width="match_parent"
       android:layout_height="match_parent"
       android:padding="16dp">
       
       <LinearLayout
           android:layout_width="match_parent"
           android:layout_height="wrap_content"
           android:orientation="vertical"
           android:padding="8dp">
           
           <com.google.android.material.textfield.TextInputLayout
               android:layout_width="match_parent"
               android:layout_height="wrap_content"
               android:layout_marginBottom="8dp"
               android:hint="Title">
               
               <com.google.android.material.textfield.TextInputEditText
                   android:id="@+id/editTextTitle"
                   android:layout_width="match_parent"
                   android:layout_height="wrap_content"
                   android:inputType="textCapSentences"
                   android:maxLines="1"/>
           </com.google.android.material.textfield.TextInputLayout>
           
           <com.google.android.material.textfield.TextInputLayout
               android:layout_width="match_parent"
               android:layout_height="wrap_content"
               android:layout_marginBottom="8dp"
               android:hint="Description (optional)">
               
               <com.google.android.material.textfield.TextInputEditText
                   android:id="@+id/editTextDescription"
                   android:layout_width="match_parent"
                   android:layout_height="wrap_content"
                   android:inputType="textCapSentences"
                   android:maxLines="3"/>
           </com.google.android.material.textfield.TextInputLayout>
           
           <com.google.android.material.textfield.TextInputLayout
               android:layout_width="match_parent"
               android:layout_height="wrap_content"
               android:layout_marginBottom="8dp"
               android:hint="Due Date (optional)">
               
               <com.google.android.material.textfield.TextInputEditText
                   android:id="@+id/editTextDueDate"
                   android:layout_width="match_parent"
                   android:layout_height="wrap_content"
                   android:focusable="false"
                   android:inputType="none"/>
           </com.google.android.material.textfield.TextInputLayout>
           
           <TextView
               android:layout_width="wrap_content"
               android:layout_height="wrap_content"
               android:layout_marginBottom="8dp"
               android:text="Priority"
               android:textAppearance="?attr/textAppearanceSubtitle1"/>
           
           <RadioGroup
               android:id="@+id/radioGroupPriority"
               android:layout_width="match_parent"
               android:layout_height="wrap_content"
               android:layout_marginBottom="16dp"
               android:orientation="horizontal">
               
               <RadioButton
                   android:id="@+id/radioButtonLow"
                   android:layout_width="0dp"
                   android:layout_height="wrap_content"
                   android:layout_weight="1"
                   android:text="Low"/>
               
               <RadioButton
                   android:id="@+id/radioButtonMedium"
                   android:layout_width="0dp"
                   android:layout_height="wrap_content"
                   android:layout_weight="1"
                   android:text="Medium"/>
               
               <RadioButton
                   android:id="@+id/radioButtonHigh"
                   android:layout_width="0dp"
                   android:layout_height="wrap_content"
                   android:layout_weight="1"
                   android:text="High"/>
           </RadioGroup>
           
       </LinearLayout>
   </ScrollView>
   ```

## Step 6: Implement RecyclerView Adapter

```kotlin
class TodoAdapter(
    private val onTodoClick: (Todo) -> Unit,
    private val onTodoCheck: (Todo, Boolean) -> Unit
) : ListAdapter<Todo, TodoAdapter.TodoViewHolder>(DiffCallback()) {

    class TodoViewHolder(private val binding: ItemTodoBinding) : RecyclerView.ViewHolder(binding.root) {
        fun bind(todo: Todo, onTodoClick: (Todo) -> Unit, onTodoCheck: (Todo, Boolean) -> Unit) {
            binding.textViewTitle.text = todo.title
            binding.textViewDescription.text = todo.description
            binding.checkBoxCompleted.isChecked = todo.isCompleted
            
            // Format due date
            todo.dueDate?.let {
                val dateFormat = SimpleDateFormat("MMM dd, yyyy", Locale.getDefault())
                binding.textViewDueDate.text = dateFormat.format(Date(it))
                binding.textViewDueDate.visibility = View.VISIBLE
            } ?: run {
                binding.textViewDueDate.visibility = View.GONE
            }
            
            // Set priority icon
            val priorityIcon = when (todo.priority) {
                2 -> R.drawable.ic_priority_high
                1 -> R.drawable.ic_priority_medium
                else -> R.drawable.ic_priority_low
            }
            binding.imageViewPriority.setImageResource(priorityIcon)
            
            // Set click listeners
            binding.root.setOnClickListener { onTodoClick(todo) }
            binding.checkBoxCompleted.setOnCheckedChangeListener { _, isChecked ->
                onTodoCheck(todo, isChecked)
            }
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): TodoViewHolder {
        val binding = ItemTodoBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return TodoViewHolder(binding)
    }

    override fun onBindViewHolder(holder: TodoViewHolder, position: Int) {
        val currentTodo = getItem(position)
        holder.bind(currentTodo, onTodoClick, onTodoCheck)
    }

    class DiffCallback : DiffUtil.ItemCallback<Todo>() {
        override fun areItemsTheSame(oldItem: Todo, newItem: Todo): Boolean {
            return oldItem.id == newItem.id
        }

        override fun areContentsTheSame(oldItem: Todo, newItem: Todo): Boolean {
            return oldItem == newItem
        }
    }
}
```

## Step 7: Implement Main Activity

```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding
    private lateinit var todoViewModel: TodoViewModel
    private lateinit var todoAdapter: TodoAdapter

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        // Initialize ViewModel
        todoViewModel = ViewModelProvider(this).get(TodoViewModel::class.java)

        // Setup RecyclerView
        setupRecyclerView()

        // Observe todos
        todoViewModel.allTodos.observe(this) { todos ->
            todos?.let { todoAdapter.submitList(it) }
        }

        // Set click listeners
        binding.fabAdd.setOnClickListener {
            showAddEditTodoDialog(null)
        }
    }

    private fun setupRecyclerView() {
        todoAdapter = TodoAdapter(
            onTodoClick = { todo ->
                showAddEditTodoDialog(todo)
            },
            onTodoCheck = { todo, isChecked ->
                val updatedTodo = todo.copy(isCompleted = isChecked)
                todoViewModel.update(updatedTodo)
            }
        )

        binding.recyclerViewTodos.apply {
            adapter = todoAdapter
            layoutManager = LinearLayoutManager(this@MainActivity)
            addItemDecoration(DividerItemDecoration(this@MainActivity, DividerItemDecoration.VERTICAL))
        }
    }

    private fun showAddEditTodoDialog(todo: Todo?) {
        val dialog = AlertDialog.Builder(this)
            .setTitle(if (todo == null) "Add Todo" else "Edit Todo")
            .setView(R.layout.dialog_add_edit_todo)
            .setPositiveButton(if (todo == null) "Add" else "Update", null)
            .setNegativeButton("Cancel", null)
            .create()

        dialog.setOnShowListener {
            val positiveButton = dialog.getButton(AlertDialog.BUTTON_POSITIVE)
            val editTextTitle = dialog.findViewById<TextInputEditText>(R.id.editTextTitle)!!
            val editTextDescription = dialog.findViewById<TextInputEditText>(R.id.editTextDescription)!!
            val editTextDueDate = dialog.findViewById<TextInputEditText>(R.id.editTextDueDate)!!
            val radioGroupPriority = dialog.findViewById<RadioGroup>(R.id.radioGroupPriority)!!

            // Set initial values if editing
            if (todo != null) {
                editTextTitle.setText(todo.title)
                editTextDescription.setText(todo.description)
                
                todo.dueDate?.let {
                    val dateFormat = SimpleDateFormat("MMM dd, yyyy", Locale.getDefault())
                    editTextDueDate.setText(dateFormat.format(Date(it)))
                }
                
                when (todo.priority) {
                    0 -> radioGroupPriority.check(R.id.radioButtonLow)
                    1 -> radioGroupPriority.check(R.id.radioButtonMedium)
                    2 -> radioGroupPriority.check(R.id.radioButtonHigh)
                }
            }

            // Date picker
            editTextDueDate.setOnClickListener {
                val calendar = Calendar.getInstance()
                DatePickerDialog(
                    this,
                    { _, year, month, day ->
                        calendar.set(year, month, day)
                        val dateFormat = SimpleDateFormat("MMM dd, yyyy", Locale.getDefault())
                        editTextDueDate.setText(dateFormat.format(calendar.time))
                    },
                    calendar.get(Calendar.YEAR),
                    calendar.get(Calendar.MONTH),
                    calendar.get(Calendar.DAY_OF_MONTH)
                ).show()
            }

            positiveButton.setOnClickListener {
                val title = editTextTitle.text.toString().trim()
                if (title.isEmpty()) {
                    editTextTitle.error = "Title cannot be empty"
                    return@setOnClickListener
                }

                val description = editTextDescription.text.toString().trim()
                val dueDate = if (editTextDueDate.text.toString().isNotEmpty()) {
                    val dateFormat = SimpleDateFormat("MMM dd, yyyy", Locale.getDefault())
                    dateFormat.parse(editTextDueDate.text.toString())?.time
                } else null

                val priority = when (radioGroupPriority.checkedRadioButtonId) {
                    R.id.radioButtonMedium -> 1
                    R.id.radioButtonHigh -> 2
                    else -> 0
                }

                if (todo == null) {
                    // Add new todo
                    val newTodo = Todo(
                        title = title,
                        description = if (description.isEmpty()) null else description,
                        dueDate = dueDate,
                        priority = priority
                    )
                    todoViewModel.insert(newTodo)
                } else {
                    // Update existing todo
                    val updatedTodo = todo.copy(
                        title = title,
                        description = if (description.isEmpty()) null else description,
                        dueDate = dueDate,
                        priority = priority
                    )
                    todoViewModel.update(updatedTodo)
                }

                dialog.dismiss()
            }
        }

        dialog.show()
    }
}
```

## Step 8: Add Additional Features

1. **Swipe to Delete**
   - Add dependency to build.gradle:
     ```gradle
     implementation "androidx.recyclerview:recyclerview:1.3.2"
     ```
   - Add ItemTouchHelper to MainActivity:
     ```kotlin
     private fun setupRecyclerView() {
         // ... existing code ...
         
         val swipeHandler = object : ItemTouchHelper.SimpleCallback(0, ItemTouchHelper.LEFT or ItemTouchHelper.RIGHT) {
             override fun onMove(recyclerView: RecyclerView, viewHolder: RecyclerView.ViewHolder, target: RecyclerView.ViewHolder): Boolean {
                 return false
             }

             override fun onSwiped(viewHolder: RecyclerView.ViewHolder, direction: Int) {
                 val position = viewHolder.adapterPosition
                 val todo = todoAdapter.currentList[position]
                 todoViewModel.delete(todo)
             }
         }
         
         ItemTouchHelper(swipeHandler).attachToRecyclerView(binding.recyclerViewTodos)
     }
     ```

2. **Confirmation Dialog for Delete**
   ```kotlin
   private fun showDeleteConfirmationDialog(todo: Todo) {
       AlertDialog.Builder(this)
           .setTitle("Delete Todo")
           .setMessage("Are you sure you want to delete this todo?")
           .setPositiveButton("Delete") { _, _ ->
               todoViewModel.delete(todo)
           }
           .setNegativeButton("Cancel") { dialog, _ ->
               dialog.dismiss()
               todoAdapter.notifyDataSetChanged() // Reset swipe
           }
           .show()
   }
   ```

3. **Filtering/Sorting Options**
   - Add menu to res/menu/menu_main.xml:
     ```xml
     <?xml version="1.0" encoding="utf-8"?>
     <menu xmlns:android="http://schemas.android.com/apk/res/android">
         <item
             android:id="@+id/action_filter"
             android:title="Filter"
             android:icon="@drawable/ic_filter"
             android:showAsAction="ifRoom"/>
     </menu>
     ```
   - Update MainActivity:
     ```kotlin
     override fun onCreateOptionsMenu(menu: Menu): Boolean {
         menuInflater.inflate(R.menu.menu_main, menu)
         return true
     }

     override fun onOptionsItemSelected(item: MenuItem): Boolean {
         return when (item.itemId) {
             R.id.action_filter -> {
                 showFilterDialog()
                 true
             }
             else -> super.onOptionsItemSelected(item)
         }
     }

     private fun showFilterDialog() {
         val items = arrayOf("All", "Completed", "Not Completed", "High Priority")
         var currentFilter = 0 // Default to All
         
         AlertDialog.Builder(this)
             .setTitle("Filter Todos")
             .setSingleChoiceItems(items, currentFilter) { _, which ->
                 currentFilter = which
             }
             .setPositiveButton("Apply") { _, _ ->
                 applyFilter(currentFilter)
             }
             .setNegativeButton("Cancel", null)
             .show()
     }

     private fun applyFilter(filter: Int) {
         when (filter) {
             0 -> todoViewModel.allTodos // All
             1 -> todoViewModel.getCompletedTodos() // Completed
             2 -> todoViewModel.getNotCompletedTodos() // Not Completed
             3 -> todoViewModel.getHighPriorityTodos() // High Priority
         }.observe(this) { todos ->
             todos?.let { todoAdapter.submitList(it) }
         }
     }
     ```
   - Add these methods to TodoViewModel:
     ```kotlin
     fun getCompletedTodos(): LiveData<List<Todo>> {
         return repository.getCompletedTodos().asLiveData()
     }
     
     fun getNotCompletedTodos(): LiveData<List<Todo>> {
         return repository.getNotCompletedTodos().asLiveData()
     }
     
     fun getHighPriorityTodos(): LiveData<List<Todo>> {
         return repository.getHighPriorityTodos().asLiveData()
     }
     ```
   - Add these queries to TodoDao:
     ```kotlin
     @Query("SELECT * FROM todos WHERE is_completed = 1 ORDER BY priority DESC, due_date ASC")
     fun getCompletedTodos(): Flow<List<Todo>>
     
     @Query("SELECT * FROM todos WHERE is_completed = 0 ORDER BY priority DESC, due_date ASC")
     fun getNotCompletedTodos(): Flow<List<Todo>>
     
     @Query("SELECT * FROM todos WHERE priority = 2 ORDER BY is_completed ASC, due_date ASC")
     fun getHighPriorityTodos(): Flow<List<Todo>>
     ```

## Step 9: Add Notifications for Due Dates

1. **Add WorkManager dependency**
   ```gradle
   implementation "androidx.work:work-runtime-ktx:2.8.1"
   ```

2. **Create NotificationWorker**
   ```kotlin
   class NotificationWorker(context: Context, params: WorkerParameters) : Worker(context, params) {
       override fun doWork(): Result {
           val todoTitle = inputData.getString("title") ?: return Result.failure()
           val todoId = inputData.getInt("id", -1)
           
           if (todoId == -1) return Result.failure()
           
           val notificationManager = applicationContext.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
           
           if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
               val channel = NotificationChannel(
                   "todo_channel",
                   "Todo Reminders",
                   NotificationManager.IMPORTANCE_DEFAULT
               )
               notificationManager.createNotificationChannel(channel)
           }
           
           val intent = Intent(applicationContext, MainActivity::class.java).apply {
               flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
           }
           
           val pendingIntent = PendingIntent.getActivity(
               applicationContext,
               0,
               intent,
               PendingIntent.FLAG_IMMUTABLE
           )
           
           val notification = NotificationCompat.Builder(applicationContext, "todo_channel")
               .setSmallIcon(R.drawable.ic_notification)
               .setContentTitle("Todo Due Soon")
               .setContentText("Don't forget: $todoTitle")
               .setPriority(NotificationCompat.PRIORITY_DEFAULT)
               .setContentIntent(pendingIntent)
               .setAutoCancel(true)
               .build()
           
           notificationManager.notify(todoId, notification)
           
           return Result.success()
       }
   }
   ```

3. **Schedule notifications when setting due dates**
   ```kotlin
   private fun scheduleNotification(todo: Todo) {
       todo.dueDate?.let { dueDate ->
           // Schedule notification 1 hour before due date
           val notificationTime = dueDate - TimeUnit.HOURS.toMillis(1)
           val currentTime = System.currentTimeMillis()
           
           if (notificationTime > currentTime) {
               val delay = notificationTime - currentTime
               
               val data = Data.Builder()
                   .putString("title", todo.title)
                   .putInt("id", todo.id)
                   .build()
               
               val notificationWork = OneTimeWorkRequestBuilder<NotificationWorker>()
                   .setInitialDelay(delay, TimeUnit.MILLISECONDS)
                   .setInputData(data)
                   .build()
               
               WorkManager.getInstance(this).enqueue(notificationWork)
           }
       }
   }
   ```

## Step 10: Final Touches

1. **Add App Theme and Colors**
   - Update colors.xml:
     ```xml
     <resources>
         <color name="purple_500">#FF6200EE</color>
         <color name="purple_700">#FF3700B3</color>
         <color name="teal_200">#FF03DAC5</color>
         <color name="teal_700">#FF018786</color>
         <color name="black">#FF000000</color>
         <color name="white">#FFFFFFFF</color>
         
         <!-- Priority colors -->
         <color name="priority_high">#FF5252</color>
         <color name="priority_medium">#FFC107</color>
         <color name="priority_low">#4CAF50</color>
     </resources>
     ```

2. **Add Icons**
   - Add appropriate vector assets for:
     - ic_add (FAB)
     - ic_filter (menu)
     - ic_priority_high, ic_priority_medium, ic_priority_low
     - ic_notification

3. **Add App Icon**
   - Update ic_launcher with your custom icon

## Step 11: Testing

1. **Unit Tests**
   - Test ViewModel logic
   - Test Repository methods
   - Test Room database operations

2. **Instrumentation Tests**
   - Test UI interactions
   - Test navigation
   - Test database operations on device

## Step 12: Build and Run

1. Connect an Android device or start an emulator
2. Click "Run" in Android Studio
3. Test all features:
   - Add, edit, delete todos
   - Mark todos as complete
   - Filter todos
   - Set due dates and receive notifications

## Additional Improvements

1. **Cloud Sync**
   - Add Firebase Firestore for cloud synchronization
   - Implement offline-first approach

2. **Categories/Tags**
   - Add ability to categorize todos

3. **Dark Mode Support**
   - Update themes for dark mode compatibility

4. **Widget**
   - Add home screen widget for quick access

5. **Backup/Restore**
   - Implement backup to Google Drive

This comprehensive to-do list app includes all the essential features and provides a solid foundation for further enhancements. The architecture follows modern Android development best practices with ViewModel, LiveData, Room, and coroutines for database operations.