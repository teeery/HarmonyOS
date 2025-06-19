# Technical Document: SQLite Database Storage in HarmonyOS Application Development

## 1. Overview

HarmonyOS, as a distributed operating system for all scenarios, has an increasing demand for local data storage in its application development. To meet this demand, HarmonyOS provides a set of RDB (Relational Database) capabilities based on SQLite. RDB deeply integrates the powerful features and stability of the SQLite database at its core, and on this basis, encapsulates and optimizes it to provide developers with more convenient and efficient data operation interfaces. This means that developers can use the high-level APIs provided by RDB for abstract data operations, or directly use native SQL statements for more fine-grained control when necessary.

The design goal of RDB is to provide a secure, efficient, and easy-to-use local data storage solution for HarmonyOS applications. It not only supports traditional relational data models but also provides potential extension capabilities for data synchronization and consistency in distributed environments, although this document primarily focuses on local storage. Through RDB, applications can persistently store user data, configuration information, cached data, etc., thereby improving user experience and application performance.

## 2. Features and Advantages of RDB Storage

HarmonyOS's RDB storage mechanism inherits many advantages of SQLite and, combined with the characteristics of the HarmonyOS ecosystem, exhibits the following significant features and advantages:

*   **Solid Foundation Based on SQLite**:
    The underlying implementation of RDB is the mature and widely used SQLite database. SQLite, with its lightweight, high-performance, serverless, and zero-configuration characteristics, is an ideal choice for local data storage on mobile and embedded devices. This means developers can leverage their familiarity with SQL language and the rich tools and resources in the SQLite ecosystem.

*   **Concept of RDB Store**:
    In the context of HarmonyOS, a relational database instance is referred to as an "RDB Store." Each RDB Store represents an independent database file, usually associated with a specific application or module. This design helps achieve data isolation and management, ensuring that data between different applications does not interfere with each other, complying with HarmonyOS's sandbox security mechanism.

*   **Highly Encapsulated API Layer**:
    HarmonyOS RDB provides a set of high-level JavaScript/TypeScript APIs that greatly simplify database operations. These APIs abstract the underlying SQL statements and complex database connection management, allowing developers to perform data insertion, updating, deletion, and querying in an object-oriented manner. For example, through objects like `ValuesBucket` and `RdbPredicates`, developers can construct complex data operation logic without directly concatenating SQL strings, thereby reducing the risk of SQL injection and improving code readability.

*   **Flexible Support for SQL and Non-SQL Operations**:
    RDB does not completely hide SQL. For scenarios requiring complex queries, transaction management, or specific database maintenance operations, developers can still directly execute native SQL statements through methods like `executeSql` or `querySql`. This flexibility allows RDB to meet the needs of rapid development while also addressing challenges of high performance and complex logic.

*   **Asynchronous Operation Mode**:
    The HarmonyOS RDB API design follows an asynchronous programming paradigm, supporting both Callback and Promise asynchronous handling methods. This is crucial for mobile application development because database operations are typically time-consuming I/O operations. Asynchronous processing can prevent blocking the UI thread, ensuring the application's fluidity and responsiveness. Developers can choose the appropriate asynchronous mode based on project requirements and personal preferences.

*   **Lightweight and High Performance**:
    RDB inherits the lightweight characteristics of SQLite, with its runtime library occupying few resources and having a fast startup speed. At the same time, optimized by HarmonyOS, RDB performs excellently in data read and write performance, meeting the performance requirements of most applications for local data storage.

*   **Data Security and Isolation**:
    HarmonyOS's sandbox mechanism provides each application with an independent data storage space. This means that an RDB Store created by one application can only be accessed by that application itself, effectively preventing data leakage and malicious tampering. In addition, RDB also supports security features such as database encryption to further protect sensitive data.

*   **Version Management and Data Migration**:
    RDB provides database version management capabilities. When an application upgrade causes changes in the database structure, developers can specify a new database version number and implement data migration logic (such as adding, deleting, or modifying table structures or data) in the callback, ensuring smooth transition and compatibility of user data.

These features collectively constitute the powerful capabilities of HarmonyOS RDB storage, making it the preferred solution for local data persistence in HarmonyOS application development.



## 3. Detailed Core API Introduction and Usage

HarmonyOS RDB module provides a series of core APIs for managing the lifecycle of RDB Stores and performing CRUD operations on their data. Understanding the detailed usage and parameters of these APIs is key to efficient development.

### 3.1 Creating, Obtaining, and Deleting an RDB Store

An RDB Store is an instance of a relational database in HarmonyOS. Before performing data operations with the database, an RDB Store instance must first be obtained.

#### 3.1.1 Obtaining an RDB Store

Obtaining an RDB Store is the first step in all database operations. HarmonyOS provides two main ways to obtain an RDB Store: callback method and Promise method. Both methods accept `Context`, `StoreConfig`, and `version` as parameters.

*   **`getRdbStore(context: Context, config: StoreConfig, version: number, callback: AsyncCallback<RdbStore>): void`**
    *   **`context`**: Represents the application context, usually obtained via `this.context`. It provides an entry point to access application environment information and is the basis for various system service operations.
    *   **`config`**: A `StoreConfig` object used to configure the RDB Store. The most important property is `name`, which specifies the name of the database file (e.g., `‘mydb.db’`). The database file will be stored in the application's private data directory. Other optional configurations may include database encryption settings, etc.
    *   **`version`**: The database version number, a positive integer. When the application first creates the database, this version number will be used. If the application is subsequently upgraded and the database structure needs to be changed, increasing this version number will trigger the database upgrade callback (e.g., `onUpgrade`), thereby enabling data migration and compatibility handling. This is a key mechanism for database version management.
    *   **`callback`**: An asynchronous callback function that is invoked when the RDB Store is successfully obtained or fails. On success, the callback function receives an `RdbStore` instance; on failure, it receives error information. This method is suitable for traditional callback programming patterns.

*   **`getRdbStore(context: Context, config: StoreConfig, version: number): Promise<RdbStore>`**
    *   The parameters are the same as the callback method, but this method returns a `Promise` object. Developers can use `async/await` syntax to handle asynchronous operations, making the code more readable and synchronous. This is the recommended asynchronous programming pattern in HarmonyOS.

**Example Code Snippet (Obtaining an RDB Store)**:

```javascript
import rdb from '@ohos.data.rdb';
import { Context } from '@ohos.ability.context';

const STORE_CONFIG = {
    name: 'my_application_data.db' // Database file name
};
const DATABASE_VERSION = 1;
let rdbStore: rdb.RdbStore | null = null;

async function initializeDatabase(appContext: Context) {
    try {
        rdbStore = await rdb.getRdbStore(appContext, STORE_CONFIG, DATABASE_VERSION);
        console.info('RDB Store initialized successfully.');
        // After successful database initialization, table structures are usually created here
        await rdbStore.executeSql('CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL, email TEXT UNIQUE)');
        console.info('Table users table created or already exists.');

        // Check database version and handle upgrades
        const currentVersion = await rdbStore.getVersion();
        if (currentVersion < DATABASE_VERSION) {
            // Execute upgrade logic
            console.warn(`Database version mismatch. Upgrading from ${currentVersion} to ${DATABASE_VERSION}`);
            // In a real project, this would call the onUpgrade method or custom upgrade logic
            // await rdbStore.executeSql("ALTER TABLE users ADD COLUMN phone TEXT");
            await rdbStore.setVersion(DATABASE_VERSION);
            console.info("Database upgraded successfully.");
        }

    } catch (err) {
        console.error(`Failed to initialize RDB Store: ${JSON.stringify(err)}`);
    }
}

// Assuming this is called in an Ability or Page
// initializeDatabase(getContext()); // Actual call needs to pass the correct Context
```

#### 3.1.2 Deleting an RDB Store

When an application no longer needs an RDB Store, it can be deleted to free up storage space. The deletion operation also supports both callback and Promise methods.

*   **`deleteRdbStore(context: Context, name: string, callback: AsyncCallback<void>): void`**
    *   **`context`**: Application context.
    *   **`name`**: The name of the database file to be deleted (e.g., `‘my_application_data.db’`).
    *   **`callback`**: Asynchronous callback function, invoked when deletion is successful or fails.

*   **`deleteRdbStore(context: Context, name: string): Promise<void>`**
    *   Parameters are the same as the callback method, returning a `Promise` object.

**Example Code Snippet (Deleting an RDB Store)**:

```javascript
async function deleteDatabase(appContext: Context, dbName: string) {
    try {
        await rdb.deleteRdbStore(appContext, dbName);
        console.info(`RDB Store '${dbName}' deleted successfully.`);
    } catch (err) {
        console.error(`Failed to delete RDB Store '${dbName}': ${JSON.stringify(err)}`);
    }
}

// Assuming this is called in an Ability or Page
// deleteDatabase(getContext(), 'my_application_data.db');
```

### 3.2 Data Management in an RDB Store: CRUD Operations

Once an RDB Store instance is obtained, data operations can be performed on it. RDB provides a set of intuitive APIs for inserting, updating, deleting, and querying data.

#### 3.2.1 Inserting Data (Insert)

The `insert` method is used to insert a new row of data into a database table. It accepts the table name and a `ValuesBucket` object as parameters.

*   **`insert(name: string, values: ValuesBucket, callback: AsyncCallback<number>): void`**
*   **`insert(name: string, values: ValuesBucket): Promise<number>`**
    *   **`name`**: The name of the target table.
    *   **`values`**: A `ValuesBucket` object, a collection of key-value pairs where keys are column names and values are the data to be inserted. `ValuesBucket` is similar to `ContentValues` in Java or `Bundle` in Android.
    *   **Return Value**: On successful insertion, returns the row ID of the newly inserted row (usually the primary key ID). If insertion fails, returns -1.

**Creating and Using `ValuesBucket`**:
`ValuesBucket` is a simple JavaScript object used to encapsulate data to be inserted or updated. For example: `{ 'column_name1': value1, 'column_name2': value2 }`.

**Example Code Snippet (Inserting Data)**:

```javascript
import rdb from '@ohos.data.rdb';

interface User {
    name: string;
    email: string;
    age?: number; // Optional field
}

async function insertUser(user: User) {
    if (!rdbStore) {
        console.error('RDB Store is not initialized.');
        return;
    }
    const valuesBucket: rdb.ValuesBucket = {
        'name': user.name,
        'email': user.email
    };
    if (user.age !== undefined) {
        valuesBucket['age'] = user.age;
    }

    try {
        const rowId = await rdbStore.insert('users', valuesBucket);
        console.info(`User '${user.name}' inserted with row ID: ${rowId}`);
    } catch (err) {
        console.error(`Failed to insert user '${user.name}': ${JSON.stringify(err)}`);
    }
}

// Example calls
// insertUser({ name: 'Alice', email: 'alice@example.com', age: 30 });
// insertUser({ name: 'Bob', email: 'bob@example.com' });
```

#### 3.2.2 Updating Data (Update)

The `update` method is used to modify data in a database table that meets specific conditions. It requires `ValuesBucket` (new data) and `RdbPredicates` (update conditions) as parameters.

*   **`update(values: ValuesBucket, rdbPredicates: RdbPredicates, callback: AsyncCallback<number>): void`**
*   **`update(values: ValuesBucket, rdbPredicates: RdbPredicates): Promise<number>`**
    *   **`values`**: A `ValuesBucket` object containing the columns to be updated and their new values.
    *   **`rdbPredicates`**: An `RdbPredicates` object used to define the conditions for the update operation. Only rows that satisfy these conditions will be updated. If `rdbPredicates` is empty or no conditions are specified, all rows may be updated (use with caution).
    *   **Return Value**: On successful update, returns the number of affected rows. If the update fails or no rows meet the conditions, returns 0.

**Creating and Using `RdbPredicates`**:
`RdbPredicates` is central to building query or update conditions. It provides a series of methods to construct various conditions found in SQL `WHERE` clauses, such as `equalTo`, `notEqualTo`, `greaterThan`, `lessThan`, `like`, `in`, etc.

**Example Code Snippet (Updating Data)**:

```javascript
import rdb from '@ohos.data.rdb';

async function updateUserEmail(oldEmail: string, newEmail: string) {
    if (!rdbStore) {
        console.error('RDB Store is not initialized.');
        return;
    }
    const valuesBucket: rdb.ValuesBucket = {
        'email': newEmail
    };
    const predicates = new rdb.RdbPredicates('users');
    predicates.equalTo('email', oldEmail); // Update records where email is oldEmail

    try {
        const rowsAffected = await rdbStore.update(valuesBucket, predicates);
        console.info(`Updated ${rowsAffected} row(s) for email '${oldEmail}'.`);
    } catch (err) {
        console.error(`Failed to update user email: ${JSON.stringify(err)}`);
    }
}

// Example calls
// updateUserEmail('alice@example.com', 'alice.new@example.com');
```

#### 3.2.3 Deleting Data (Delete)

The `delete` method is used to delete data from a database table that meets specific conditions. It requires `RdbPredicates` as a parameter.

*   **`delete(rdbPredicates: RdbPredicates, callback: AsyncCallback<number>): void`**
*   **`delete(rdbPredicates: RdbPredicates): Promise<number>`**
    *   **`rdbPredicates`**: An `RdbPredicates` object used to define the conditions for the delete operation. Only rows that satisfy these conditions will be deleted. If `rdbPredicates` is empty or no conditions are specified, all rows may be deleted (use with caution).
    *   **Return Value**: On successful deletion, returns the number of affected rows. If the deletion fails or no rows meet the conditions, returns 0.

**Example Code Snippet (Deleting Data)**:

```javascript
import rdb from '@ohos.data.rdb';

async function deleteUserByName(name: string) {
    if (!rdbStore) {
        console.error('RDB Store is not initialized.');
        return;
    }
    const predicates = new rdb.RdbPredicates('users');
    predicates.equalTo('name', name); // Delete records where name is the specified value

    try {
        const rowsAffected = await rdbStore.delete(predicates);
        console.info(`Deleted ${rowsAffected} row(s) for user '${name}'.`);
    } catch (err) {
        console.error(`Failed to delete user: ${JSON.stringify(err)}`);
    }
}

// Example calls
// deleteUserByName('Bob');
```

#### 3.2.4 Querying Data (Query)

Querying is one of the most frequently used database operations. RDB provides two ways to query: conditional queries using `RdbPredicates` and direct SQL statement queries.

*   **Querying using `RdbPredicates`**:
    *   **`query(rdbPredicates: RdbPredicates, columns: Array<string>, callback: AsyncCallback<ResultSet>): void`**
    *   **`query(rdbPredicates: RdbPredicates, columns: Array<string>): Promise<ResultSet>`**
        *   **`rdbPredicates`**: An `RdbPredicates` object used to define query conditions. Can include `where` clauses, `orderBy`, `limit`, `offset`, etc.
        *   **`columns`**: An array of strings specifying the column names to query. If an empty array is passed or not specified, all columns will be queried.
        *   **Return Value**: On successful query, returns a `ResultSet` object. A `ResultSet` is a cursor used to iterate through the query results. Be sure to call `close()` to release resources when finished.

**Iterating and Extracting Data from `ResultSet`**:
`ResultSet` provides methods such as `goToFirstRow()`, `goToNextRow()`, `getColumnIndex(columnName: string)`, `getString(columnIndex: number)`, `getLong(columnIndex: number)`, etc., to iterate through the result set and extract data.

**Example Code Snippet (Querying Data using `RdbPredicates`)**:

```javascript
import rdb from '@ohos.data.rdb';

async function queryUsersByAge(minAge: number, maxAge: number) {
    if (!rdbStore) {
        console.error('RDB Store is not initialized.');
        return;
    }
    const predicates = new rdb.RdbPredicates('users');
    predicates.greaterThanOrEqualTo('age', minAge)
              .lessThanOrEqualTo('age', maxAge)
              .orderByAsc('name'); // Order by name in ascending order

    try {
        const resultSet = await rdbStore.query(predicates, ['name', 'email', 'age']);
        console.info(`Query results for users between ${minAge} and ${maxAge} years old:`);
        if (resultSet.rowCount > 0 && resultSet.goToFirstRow()) {
            do {
                const name = resultSet.getString(resultSet.getColumnIndex('name'));
                const email = resultSet.getString(resultSet.getColumnIndex('email'));
                const age = resultSet.getLong(resultSet.getColumnIndex('age'));
                console.info(`  Name: ${name}, Email: ${email}, Age: ${age}`);
            } while (resultSet.goToNextRow());
        } else {
            console.info('  No users found matching the criteria.');
        }
        resultSet.close(); // Must close ResultSet
    } catch (err) {
        console.error(`Failed to query users: ${JSON.stringify(err)}`);
    }
}

// Example calls
// queryUsersByAge(20, 35);
```

*   **Directly Executing SQL Statements for Querying**:
    *   **`querySql(sql: string, bindArgs: Array<ValueType>, callback: AsyncCallback<ResultSet>): void`**
    *   **`querySql(sql: string, bindArgs?: Array<ValueType>): Promise<ResultSet>`**
        *   **`sql`**: The SQL query statement to execute. Placeholders `?` can be used to prevent SQL injection.
        *   **`bindArgs`**: An array containing values to replace the placeholders in the SQL statement. The order must match the placeholders.
        *   **Return Value**: On successful query, returns a `ResultSet` object.

**Example Code Snippet (Querying Data using SQL Statements)**:

```javascript
import rdb from '@ohos.data.rdb';

async function queryUsersBySql(minAge: number) {
    if (!rdbStore) {
        console.error('RDB Store is not initialized.');
        return;
    }
    const sql = 'SELECT name, email, age FROM users WHERE age > ? ORDER BY name DESC';
    const bindArgs = [minAge];

    try {
        const resultSet = await rdbStore.querySql(sql, bindArgs);
        console.info(`Query results for users older than ${minAge} (via SQL):`);
        if (resultSet.rowCount > 0 && resultSet.goToFirstRow()) {
            do {
                const name = resultSet.getString(resultSet.getColumnIndex('name'));
                const email = resultSet.getString(resultSet.getColumnIndex('email'));
                const age = resultSet.getLong(resultSet.getColumnIndex('age'));
                console.info(`  Name: ${name}, Email: ${email}, Age: ${age}`);
            } while (resultSet.goToNextRow());
        }
        resultSet.close();
    } catch (err) {
        console.error(`Failed to query users via SQL: ${JSON.stringify(err)}`);
    }
}

// Example calls
// queryUsersBySql(25);
```

## 4. Database Version Management and Upgrades

Throughout the application development lifecycle, database structures may evolve with changing business requirements. HarmonyOS RDB provides a comprehensive database version management mechanism to ensure smooth data migration when database structures change, preventing data loss or application crashes.

### 4.1 Role of Version Numbers

Each RDB Store has an associated version number. When an application first creates a database, an initial version number is specified. When a new version of the application is released and the new version requires changes to the database structure (e.g., adding new tables, adding new columns, modifying column types, etc.), developers need to increment the database version number.

### 4.2 Upgrade Process

When the `getRdbStore` method is called, if the `version` parameter passed is greater than the current version number of the database file, the system automatically triggers the database upgrade process. Developers need to configure the corresponding callback function in `StoreConfig` to handle the upgrade logic.

**`onUpgrade` Callback in `StoreConfig`**:

```javascript
const STORE_CONFIG_WITH_UPGRADE = {
    name: 'my_application_data.db',
    onUpgrade: (rdbStore: rdb.RdbStore, oldVersion: number, newVersion: number) => {
        console.info(`Database upgrading from version ${oldVersion} to ${newVersion}`);
        // Execute corresponding SQL statements for data migration based on old and new versions
        if (oldVersion === 1 && newVersion === 2) {
            // Logic for upgrading from version 1 to version 2
            rdbStore.executeSql('ALTER TABLE users ADD COLUMN phone TEXT');
            console.info('Added phone column to users table.');
        } else if (oldVersion === 2 && newVersion === 3) {
            // Logic for upgrading from version 2 to version 3
            rdbStore.executeSql('CREATE TABLE IF NOT EXISTS products (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL, price REAL)');
            console.info('Created products table.');
        }
        // ... more version upgrade logic
    }
};

// Use the new configuration during initialization
// rdb.getRdbStore(appContext, STORE_CONFIG_WITH_UPGRADE, NEW_DATABASE_VERSION);
```

In the `onUpgrade` callback, developers can:
*   Execute `ALTER TABLE` statements to modify existing table structures.
*   Execute `CREATE TABLE` statements to create new tables.
*   Execute `DROP TABLE` statements to delete old tables (use with caution).
*   Execute `INSERT`, `UPDATE`, `DELETE` statements to migrate or adjust data.

**Important Note**: In the `onUpgrade` callback, all database operations should be completed within a transaction to ensure data consistency. HarmonyOS RDB automatically handles transactions, but developers still need to ensure the atomicity of the logic.

### 4.3 Downgrade Handling (onDowngrade)

Although uncommon, it may sometimes be necessary to downgrade the database to an older version (e.g., when a user installs an older version of the application). `StoreConfig` also provides an `onDowngrade` callback to handle such situations. Downgrading is generally more complex than upgrading, as it may involve data loss, so special care should be taken in its design.

```javascript
const STORE_CONFIG_WITH_DOWNGRADE = {
    name: 'my_application_data.db',
    onDowngrade: (rdbStore: rdb.RdbStore, oldVersion: number, newVersion: number) => {
        console.warn(`Database downgrading from version ${oldVersion} to ${newVersion}`);
        // Downgrade logic, may need to delete tables or columns added in the newer version
        // rdbStore.executeSql('DROP TABLE IF EXISTS products');
    }
};
```

## 5. Best Practices and Important Considerations

To ensure efficient, stable, and secure HarmonyOS RDB storage, developers should follow these best practices and important considerations:

### 5.1 Permissions Management

In HarmonyOS, applications require corresponding permissions to access local storage resources. For RDB database operations, database read and write permissions need to be declared in the application's `module.json5` file.

```json
{
  // ... Other configurations
  "requestPermissions": [
    {
      "name": "ohos.permission.WRITE_DATA",
      "reason": "Need to write data to database"
    },
    {
      "name": "ohos.permission.READ_DATA",
      "reason": "Need to read data from database"
    }
  ]
}
```

Ensure that only permissions actually needed by the application are declared, and clearly explain the purpose of the permissions to the user.

### 5.2 Data Isolation and Sandbox Mechanism

HarmonyOS employs a strict sandbox mechanism to isolate application data. Each application has its independent storage space, and an RDB Store created by one application can only be accessed by that application itself. This means:
*   Data between different applications is isolated and cannot directly share database files.
*   This enhances data security, preventing malicious applications from accessing or tampering with other application's data.
*   Developers do not need to worry about data conflicts with other applications.

### 5.3 Asynchronous Operations and UI Thread

Database operations (such as querying large amounts of data, inserting multiple records) are typically time-consuming. If these operations are performed on the UI thread, it will cause UI freezes and affect user experience. Therefore, HarmonyOS RDB API is designed in an asynchronous mode, and developers should fully utilize Promise or callback mechanisms to perform database operations in a background thread and return the results to the UI thread for updates.

*   **Avoid executing time-consuming database operations directly on the UI thread.**
*   **Use `async/await` or Promise chains to manage asynchronous flows, improving code readability.**

### 5.4 Error Handling and Logging

In real-world applications, database operations may encounter various errors, such as database file corruption, insufficient permissions, SQL syntax errors, data constraint violations, etc. A robust error handling mechanism is crucial.

*   **Use `try-catch` blocks to catch Promise rejections or errors in callbacks.**
*   **Log detailed error messages**: Use HarmonyOS's logging tools (e.g., `hilog`) to record the success and failure of database operations, error messages, exception stacks, etc., for easy troubleshooting and debugging.
*   **Provide user-friendly error messages**: When user-perceivable errors occur, clear prompts should be given to guide users to solve the problem or provide feedback.

### 5.5 Transaction Management

Transactions are key to ensuring atomicity, consistency, isolation, and durability (ACID) of database operations. When a series of interrelated database operations need to be performed, they should be encapsulated within a transaction. If any step in the transaction fails, the entire transaction can be rolled back, ensuring that data does not end up in an inconsistent state.

HarmonyOS RDB provides `beginTransaction()`, `commit()`, and `rollBack()` methods to manually manage transactions.

```javascript
async function performTransaction() {
    if (!rdbStore) {
        console.error('RDB Store is not initialized.');
        return;
    }
    try {
        await rdbStore.beginTransaction();
        // Operation 1: Insert user
        await rdbStore.insert('users', { 'name': 'TransactionUser1', 'email': 'tx1@example.com' });
        // Operation 2: Update user
        const predicates = new rdb.RdbPredicates('users');
        predicates.equalTo('name', 'TransactionUser1');
        await rdbStore.update({ 'age': 25 }, predicates);

        // Simulate a potentially failing operation
        // if (someConditionFails) {
        //     throw new Error('Simulated transaction failure');
        // }

        await rdbStore.commit();
        console.info('Transaction committed successfully.');
    } catch (err) {
        await rdbStore.rollBack();
        console.error(`Transaction rolled back: ${JSON.stringify(err)}`);
    }
}

// Example calls
// performTransaction();
```

### 5.6 Database Connection and Resource Management

*   **Avoid frequently opening and closing database connections**: The `getRdbStore` operation may involve file I/O, and frequent calls will affect performance. Typically, the RDB Store instance is obtained at application startup and reused throughout the application's lifecycle.
*   **Promptly close `ResultSet`**: After using a `ResultSet` object returned by each query operation, be sure to call its `close()` method to release underlying cursor resources and prevent memory leaks.

### 5.7 SQL Injection Prevention

When using `querySql` or `executeSql` methods to execute native SQL statements, always use parameter binding (i.e., using placeholders `?` and the `bindArgs` array) to pass user input or dynamic data, rather than directly concatenating strings. This can effectively prevent SQL injection attacks.

**Incorrect Example (Vulnerable to SQL Injection)**:
```javascript
// const userName = userInputName; // Assume this is user input
// const sql = `SELECT * FROM users WHERE name = '${userName}'`; // Incorrect: Direct string concatenation
// await rdbStore.querySql(sql, []);
```

**Correct Example (Using Parameter Binding)**:
```javascript
const userName = userInputName; // Assume this is user input
const sql = `SELECT * FROM users WHERE name = ?`; // Correct: Use placeholder
await rdbStore.querySql(sql, [userName]); // Correct: Pass parameters via bindArgs
```

### 5.8 Performance Optimization

*   **Reasonable Database Schema Design**: Choose appropriate data types, create necessary indexes, and optimize table structures.
*   **Batch Operations**: For large amounts of data insertion, updating, or deletion, try to use batch operations to reduce the number of database I/O operations.
*   **Avoid Full Table Scans**: Ensure that query operations can utilize indexes through `RdbPredicates` or SQL `WHERE` clauses.

## 6. Conclusion

HarmonyOS RDB, as a core component for local data storage, provides developers with powerful and flexible SQLite database operation capabilities. Through its high-level APIs and good support for asynchronous operations, developers can efficiently build responsive, data-persistent HarmonyOS applications. Understanding the characteristics of RDB, the usage of core APIs, and adhering to best practices such as permissions management, data isolation, asynchronous processing, error handling, transaction management, and SQL injection prevention, will help developers build high-performance, highly reliable, and highly secure HarmonyOS applications.

As the HarmonyOS ecosystem continues to evolve, RDB capabilities will also continue to advance, providing developers with more powerful data management tools. Developers are advised to continue to pay attention to HarmonyOS official documentation and the community for the latest RDB development information and best practices.

## 7. References

*   [1] HarmonyOS Official Documentation - RDB Development Guidelines: [https://device.harmonyos.com/en/docs/apiref/database-relational-guidelines-0000001333800361](https://device.harmonyos.com/en/docs/apiref/database-relational-guidelines-0000001333800361)
*   [2] HarmonyOS Official Documentation - Relational Database Overview: [https://device.harmonyos.com/en/docs/apiref/doc-guides/database-relational-overview-0000000000030046](https://device.harmonyos.com/en/docs/apiref/doc-guides/database-relational-overview-0000000000030046)
*   [3] SQLite Official Website: [https://www.sqlite.org/](https://www.sqlite.org/)


