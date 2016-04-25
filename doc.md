## Overview

```python
class DatabaseVendor(db.Entity):
    name = Required(str)
    commercial = Required(bool)
    features = Optional(Json)
    supported_os = Optional(Json)
    
with db_session:
    DatabaseVendor(name='oracle', commercial=True, features={
        'clustering': True,
        'json': True,
    })
```
Notice the `Json` type, passed as a parameter to the last two entity attributes.

You can set/update a Json value as a normal Python dict/list:
```python
with db_session:
    oracle = get(d for d in DatabaseVendor if d.name == 'oracle')
    oracle.features['GUI admin'] = True
```
 
In Python queries, you can access attributes by its JSON path,
using dictionary item access, either int or str.
For example, `obj.data['items'][0]['title']` in case we have a respective Json field `data`.

Or, in the example above, we can make a select like:
```python
>>> select((d.name, d.features['clustering']) for d in DatabaseVendor)[:]
[('oracle', True)]
```


In queries, `dict` is associated with JSON type. For example, here is how you can find element
by its JSON value:
```python
get(o for o in MyEntity if o.data == {'some': 'data'})
```
The `list` type probably will be used for the array type in databases supporting it,
so currently you should wrap it in a `Json` object:

```python
with db_session:
    oracle = get(d for d in DatabaseVendor if d.name == 'oracle')
    oracle.supported_os = Json(['RHEL', 'Oracle Linux', 'SUSE'])
```

### Tracking changes

Within a Pony session, given a set of fetched entities,
Pony is tracking changes to the values of their attributes so it could flush them back
into the database. Though in Python `dict` and `list` instances are not immutable,
changes to the respective attributes can also be tracked by Pony.
To accomplish this, custom `dict` and `list` objects are returned
when Json attributes are fetched.

```python
with db_session:
    obj = Entity[1]
    # Modifying obj.data
    del obj.data['items'][2]
   # changes will be commited
```

### Length of json value

You can reference the length of JSON data in queries:

```python
select(obj for obj in MyEntity if len(obj.data) == 1)
```

`Note:` Not available in sqlite when used without the `json1` extension.

##  Database support

The common set of features supported for all databases is
- read/write operations for JSON fields
- querying JSON paths

In the examples below, data field is supposed to be of `Json` type.

### SQLite

You can benefit from better JSON support in sqlite if you have `json1` extension
installed. Otherwise, JSON values are fetched and parsed in an ineffective way.

### PostgreSQL
Besides the common set, following features are supported:

- JSON concatenation

    ```python
    select(e.data | e.extra_data for e in MyEntity)
    ```
    
    Besides an attribute, `extra_data` could be a Python dict as well.

- Contains JSON

    We can check whether a JSON contains the specified part.
    Python `in` operator is used for this:
    
    ```python
    select({"visible": True} in obj.data['details'] for obj in MyEntity)
    ```

### MySql and Oracle

MySql and Oracle support wildcards in JSON path, meaning "any value".
For example:
```python
select(o for o in MyEntity if o.data['items'][:]['visible'] == True)
```
Here we have used a wildcard for a numeric value of the item index. As you see,
Python empty slice is used for this.

Python `Ellipsis` represents a wildcard for a string JSON key, like this:
```python
select(o.data[...]['title'] for o in MyEntity)
```