# Building a CRUD Web Application with React Native and SQLite

Here's a step-by-step guide to creating a CRUD (Create, Read, Update, Delete) application using React Native and SQLite.

## Step 1: Set Up Your React Native Project

First, create a new React Native project:

```bash
npx react-native init CrudApp
cd CrudApp
```

## Step 2: Install Required Dependencies

Install SQLite and other necessary packages:

```bash
npm install @react-native-community/sqlite-storage
npm install react-native-sqlite-storage
```

For iOS, you'll need to install pods:

```bash
cd ios && pod install && cd ..
```

## Step 3: Create Database Helper File

Create a new file `src/database/Database.js`:

```javascript
import SQLite from 'react-native-sqlite-storage';

const database_name = "CrudApp.db";
const database_version = "1.0";
const database_displayname = "CRUD Application Database";
const database_size = 200000;

const db = SQLite.openDatabase(
  database_name,
  database_version,
  database_displayname,
  database_size
);

const initDB = () => {
  db.transaction((tx) => {
    tx.executeSql(
      `CREATE TABLE IF NOT EXISTS items (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT,
        description TEXT
      );`,
      [],
      () => console.log('Database and table created successfully'),
      (error) => console.log('Error creating table: ', error)
    );
  });
};

const getDB = () => {
  return db;
};

export { initDB, getDB };
```

## Step 4: Create Item Service for CRUD Operations

Create `src/services/ItemService.js`:

```javascript
import { getDB } from '../database/Database';

const db = getDB();

const addItem = (name, description, callback) => {
  db.transaction((tx) => {
    tx.executeSql(
      'INSERT INTO items (name, description) VALUES (?, ?)',
      [name, description],
      (_, result) => callback(true, result),
      (_, error) => callback(false, error)
    );
  });
};

const getItems = (callback) => {
  db.transaction((tx) => {
    tx.executeSql(
      'SELECT * FROM items',
      [],
      (_, result) => callback(true, result.rows.raw()),
      (_, error) => callback(false, error)
    );
  });
};

const updateItem = (id, name, description, callback) => {
  db.transaction((tx) => {
    tx.executeSql(
      'UPDATE items SET name = ?, description = ? WHERE id = ?',
      [name, description, id],
      (_, result) => callback(true, result),
      (_, error) => callback(false, error)
    );
  });
};

const deleteItem = (id, callback) => {
  db.transaction((tx) => {
    tx.executeSql(
      'DELETE FROM items WHERE id = ?',
      [id],
      (_, result) => callback(true, result),
      (_, error) => callback(false, error)
    );
  });
};

export { addItem, getItems, updateItem, deleteItem };
```

## Step 5: Create UI Components

### Create Item List Component (`src/components/ItemList.js`)

```javascript
import React, { useState, useEffect } from 'react';
import { View, Text, FlatList, TouchableOpacity, StyleSheet } from 'react-native';
import { getItems, deleteItem } from '../services/ItemService';

const ItemList = ({ navigation }) => {
  const [items, setItems] = useState([]);

  useEffect(() => {
    fetchItems();
  }, []);

  const fetchItems = () => {
    getItems((success, data) => {
      if (success) {
        setItems(data);
      }
    });
  };

  const handleDelete = (id) => {
    deleteItem(id, (success) => {
      if (success) {
        fetchItems();
      }
    });
  };

  const renderItem = ({ item }) => (
    <View style={styles.itemContainer}>
      <TouchableOpacity 
        onPress={() => navigation.navigate('ItemDetail', { item })}
        style={styles.itemContent}
      >
        <Text style={styles.itemName}>{item.name}</Text>
        <Text style={styles.itemDescription}>{item.description}</Text>
      </TouchableOpacity>
      <View style={styles.buttonsContainer}>
        <TouchableOpacity 
          onPress={() => navigation.navigate('EditItem', { item })}
          style={styles.editButton}
        >
          <Text style={styles.buttonText}>Edit</Text>
        </TouchableOpacity>
        <TouchableOpacity 
          onPress={() => handleDelete(item.id)}
          style={styles.deleteButton}
        >
          <Text style={styles.buttonText}>Delete</Text>
        </TouchableOpacity>
      </View>
    </View>
  );

  return (
    <FlatList
      data={items}
      renderItem={renderItem}
      keyExtractor={(item) => item.id.toString()}
    />
  );
};

const styles = StyleSheet.create({
  itemContainer: {
    padding: 15,
    borderBottomWidth: 1,
    borderBottomColor: '#ccc',
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
  },
  itemContent: {
    flex: 1,
  },
  itemName: {
    fontSize: 18,
    fontWeight: 'bold',
  },
  itemDescription: {
    fontSize: 14,
    color: '#666',
  },
  buttonsContainer: {
    flexDirection: 'row',
  },
  editButton: {
    backgroundColor: '#4CAF50',
    padding: 8,
    borderRadius: 4,
    marginRight: 5,
  },
  deleteButton: {
    backgroundColor: '#f44336',
    padding: 8,
    borderRadius: 4,
  },
  buttonText: {
    color: 'white',
  },
});

export default ItemList;
```

### Create Item Form Component (`src/components/ItemForm.js`)

```javascript
import React, { useState, useEffect } from 'react';
import { View, Text, TextInput, TouchableOpacity, StyleSheet } from 'react-native';

const ItemForm = ({ navigation, route }) => {
  const [name, setName] = useState('');
  const [description, setDescription] = useState('');
  const [isEdit, setIsEdit] = useState(false);
  const [id, setId] = useState(null);

  useEffect(() => {
    if (route.params?.item) {
      const { item } = route.params;
      setName(item.name);
      setDescription(item.description);
      setId(item.id);
      setIsEdit(true);
    }
  }, [route.params?.item]);

  const handleSubmit = () => {
    if (isEdit) {
      // Call update function
      route.params.onUpdate(id, name, description);
    } else {
      // Call create function
      route.params.onCreate(name, description);
    }
    navigation.goBack();
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>{isEdit ? 'Edit Item' : 'Add New Item'}</Text>
      
      <TextInput
        style={styles.input}
        placeholder="Name"
        value={name}
        onChangeText={setName}
      />
      
      <TextInput
        style={[styles.input, styles.descriptionInput]}
        placeholder="Description"
        value={description}
        onChangeText={setDescription}
        multiline
      />
      
      <TouchableOpacity style={styles.submitButton} onPress={handleSubmit}>
        <Text style={styles.submitButtonText}>{isEdit ? 'Update' : 'Save'}</Text>
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    padding: 20,
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    marginBottom: 20,
    textAlign: 'center',
  },
  input: {
    borderWidth: 1,
    borderColor: '#ccc',
    borderRadius: 4,
    padding: 10,
    marginBottom: 15,
  },
  descriptionInput: {
    height: 100,
    textAlignVertical: 'top',
  },
  submitButton: {
    backgroundColor: '#2196F3',
    padding: 15,
    borderRadius: 4,
    alignItems: 'center',
  },
  submitButtonText: {
    color: 'white',
    fontWeight: 'bold',
  },
});

export default ItemForm;
```

### Create Item Detail Component (`src/components/ItemDetail.js`)

```javascript
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

const ItemDetail = ({ route }) => {
  const { item } = route.params;

  return (
    <View style={styles.container}>
      <Text style={styles.title}>{item.name}</Text>
      <Text style={styles.description}>{item.description}</Text>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    padding: 20,
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    marginBottom: 20,
  },
  description: {
    fontSize: 16,
    lineHeight: 24,
  },
});

export default ItemDetail;
```

## Step 6: Set Up Navigation

Install React Navigation:

```bash
npm install @react-navigation/native @react-navigation/stack
npm install react-native-gesture-handler react-native-reanimated react-native-screens react-native-safe-area-context
```

Create `src/navigation/AppNavigator.js`:

```javascript
import React from 'react';
import { createStackNavigator } from '@react-navigation/stack';
import { NavigationContainer } from '@react-navigation/native';
import ItemList from '../components/ItemList';
import ItemForm from '../components/ItemForm';
import ItemDetail from '../components/ItemDetail';
import { addItem, updateItem } from '../services/ItemService';

const Stack = createStackNavigator();

const AppNavigator = () => {
  return (
    <NavigationContainer>
      <Stack.Navigator initialRouteName="ItemList">
        <Stack.Screen 
          name="ItemList" 
          component={ItemList} 
          options={{ title: 'Items' }}
        />
        <Stack.Screen 
          name="AddItem" 
          component={ItemForm} 
          options={{ title: 'Add Item' }}
          initialParams={{ 
            onCreate: (name, description) => {
              addItem(name, description, (success) => {
                if (!success) alert('Error adding item');
              });
            } 
          }}
        />
        <Stack.Screen 
          name="EditItem" 
          component={ItemForm} 
          options={{ title: 'Edit Item' }}
          initialParams={{ 
            onUpdate: (id, name, description) => {
              updateItem(id, name, description, (success) => {
                if (!success) alert('Error updating item');
              });
            } 
          }}
        />
        <Stack.Screen 
          name="ItemDetail" 
          component={ItemDetail} 
          options={{ title: 'Item Details' }}
        />
      </Stack.Navigator>
    </NavigationContainer>
  );
};

export default AppNavigator;
```

## Step 7: Update App.js

Modify `App.js` to initialize the database and set up navigation:

```javascript
import React, { useEffect } from 'react';
import { initDB } from './src/database/Database';
import AppNavigator from './src/navigation/AppNavigator';
import { View, Text } from 'react-native';

const App = () => {
  const [dbInitialized, setDbInitialized] = React.useState(false);

  useEffect(() => {
    initDB();
    setDbInitialized(true);
  }, []);

  if (!dbInitialized) {
    return (
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
        <Text>Initializing database...</Text>
      </View>
    );
  }

  return <AppNavigator />;
};

export default App;
```

## Step 8: Add a Floating Action Button (Optional)

To add a floating button for creating new items, modify your `ItemList.js`:

```javascript
// Add to imports
import { FloatingAction } from 'react-native-floating-action';
import Icon from 'react-native-vector-icons/MaterialIcons';

// Add to your ItemList component, before the FlatList
<>
  <FlatList ... />
  <FloatingAction
    actions={[
      {
        text: 'Add Item',
        icon: <Icon name="add" size={24} color="white" />,
        name: 'add_item',
      }
    ]}
    onPressItem={() => navigation.navigate('AddItem')}
  />
</>
```

Don't forget to install the required packages:

```bash
npm install react-native-floating-action react-native-vector-icons
```

## Step 9: Run Your Application

Now you can run your application:

For Android:
```bash
npx react-native run-android
```

For iOS:
```bash
npx react-native run-ios
```

## Additional Improvements

1. **Error Handling**: Add better error handling and user feedback
2. **Loading States**: Show loading indicators during database operations
3. **Validation**: Add form validation
4. **Search Functionality**: Implement search in the item list
5. **Sorting/Filtering**: Add options to sort or filter items
6. **Pagination**: Implement pagination for large datasets

This completes a basic CRUD application with React Native and SQLite. The app allows you to create, read, update, and delete items from a local SQLite database.