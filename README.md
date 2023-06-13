# nano-database

A postgres-like database in <100 lines of js.

```javascript
class Database {
  constructor() {
    this.tables = {};
  }

  createTable(tableName, schema) {
    this.tables[tableName] = {
      data: [],
      schema: schema,
      indexes: {},
    };
  }

  createIndex(tableName, columnName) {
    const table = this.tables[tableName];
    table.indexes[columnName] = {};

    table.data.forEach((item, index) => {
      table.indexes[columnName][item[columnName]] = index;
    });
  }

  insert(tableName, data) {
    const table = this.tables[tableName];
    const newData = {};

    Object.keys(table.schema).forEach((columnName) => {
      newData[columnName] = data[columnName] || table.schema[columnName].default;
    });

    table.data.push(newData);

    Object.keys(table.indexes).forEach((index) => {
      table.indexes[index][newData[index]] = table.data.length - 1;
    });

    return newData;
  }

  delete(tableName, conditionFn) {
    const table = this.tables[tableName];
    const indexesToDelete = [];

    table.data.forEach((item, index) => {
      if (conditionFn(item)) {
        indexesToDelete.push(index);

        Object.keys(table.indexes).forEach((indexName) => {
          delete table.indexes[indexName][item[indexName]];
        });
      }
    });

    table.data = table.data.filter((_, index) => !indexesToDelete.includes(index));
  }

  update(tableName, data, conditionFn) {
    const table = this.tables[tableName];

    table.data.forEach((item, index) => {
      if (conditionFn(item)) {
        Object.keys(data).forEach((prop) => {
          item[prop] = data[prop];

          Object.keys(table.indexes).forEach((index) => {
            if (prop === index) {
              delete table.indexes[index][item[index]];
              table.indexes[index][data[prop]] = index;
            }
          });
        });
      }
    });
  }

  search(tableName, conditionFn) {
    return this.tables[tableName].data.filter(conditionFn);
  }

  indexedSearch(tableName, columnName, value) {
    const table = this.tables[tableName];
    const indexValue = table.indexes[columnName][value];
    if (indexValue !== undefined) {
      return [table.data[indexValue]];
    } else {
      return [];
    }
  }
}

// Usage
const db = new Database();
db.createTable('users', {
  id: { type: 'int', default: null },
  name: { type: 'string', default: null },
  age: { type: 'int', default: 0 },
});

db.createIndex('users', 'id');

db.insert('users', { id: 1, name: 'Alice', age: 30 });
db.insert('users', { id: 2, name: 'Bob', age: 25 });
db.insert('users', { id: 3, name: 'Carol', age: 35 });

console.log(db.indexedSearch('users', 'id', 2)); // [{ id: 2, name: 'Bob', age: 25 }]
```
