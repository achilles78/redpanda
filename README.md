# RedPanda: Pandas & SQLAlchemy

Two great tastes that taste great together.

Use RedPanda to add simple pandas integration into your declarative models.

View [example.py](./example.py) for extended usage.

*Last Updated: `0.0.1`*


## Installation

```bash
pip install redpanda
```


## Basic Use

RedPanda wraps the `pandas.read_sql()` function into a dialect-agnostic class-method for declarative SQLAlchemy models. Use mixins to add the `redpanda()` and `redparse()` methods to your declarative model classes:

```python
# Create an in-memory SQLite database engine
engine = sqlalchemy.create_engine("sqlite://", echo=True)

# SQLAlchemy declarative base
Base = sqlalchemy.ext.declarative.declarative_base()

class MyModel(redpanda.mixins.RedPandaMixin, Base):
    # ...

MyModel.redpanda(engine)
```

Use the resulting `RedPanda` instance to transform SQLAlchemy queries into DataFrames:

```python
MyModel.redpanda(engine).frame()
```

Or parse a DataFrame into SQLAlchemy model list-generator:

```python
for model in MyModel.redparse(frame):
    print model
```


## Dialects

While the arguments to `pandas.read_sql()` are dialect-dependent, RedPanda is intended to be completely dialect-agnostic. RedPanda supports some SQLAlchemy dialects out of the box (MySQL, Postgres, and SQLite are supported). You can add support for other dialects by constructing a function to extract a parameter data-struct (eg, tuple, dict) from a compiled query statement:

```python
engine = sqlalchemy.create_engine("example://host/db")
func   = lambda statement: statement.params.items()
redpanda.dialects.add(type(engine.dialect), func)
```


## Extended Use

View [example.py](./example.py) for extended usage examples.


## RedPanda Declarative Mixin

Use the `redpanda.minxins.RedPandaMixin` mixin to add RedPanda to your declarative SQLAlchemy models.

```python
# Create an in-memory SQLite database engine
engine = sqlalchemy.create_engine("sqlite://", echo=True)

# SQLAlchemy declarative base
Base = sqlalchemy.ext.declarative.declarative_base()

# Declare our model with mixin
class Widget(redpanda.mixins.RedPandaMixin, Base):
    __tablename__ = "widgets"
    id            = sqlalchemy.Column(sqlalchemy.Integer, primary_key=True)
    timestamp     = sqlalchemy.Column(sqlalchemy.DateTime)
    name          = sqlalchemy.Column(sqlalchemy.String)
    kind          = sqlalchemy.Column(sqlalchemy.String)
    units         = sqlalchemy.Column(sqlalchemy.Integer)
```

## Mixin Customization

Add customization at the model-level by overriding the default class attributes:


#### \__redpanda__

If you wish to use your own custom `RedPanda` class, you can override the `__redpanda__` class attribute:

```python
class MyRedPanda(redpanda.orm.RedPanda):
    # ... custom logic here

class Widget(redpanda.mixins.RedPandaMixin, Base):
    # ... see above for full definition
    __redpanda__ = MyRedPanda
```


#### \__read_sql__

Set the `__read_sql__` attribute to control the defualt arguments passed to `pandas.read_sql()`

```python
class Widget(redpanda.mixins.RedPandaMixin, Base):
    # ... see above for full definition

    # Class-defined RedPanda read_sql() arguments
    # This allows us to forego passing these into Widget.redpanda()
    __read_sql__ = {
        "index_col"   : ["created_at"],
        "parse_dates" : ["created_at"] }
```


## Accessing Data

The `redpanda()` class-method accepts the following arguments:
* `engine`: A SQLAlchemy engine
* `query`: An optional SQLAlchemy query to refine the data returned by the engine
* `**read_sql`: An optional keyword-argument dict to be passed through to the `pandas.read_sql()` function. 

If the `query` argument is omitted, the default behavior is to select the entire table.

If the `**read_sql` keyword-argument dict is omitted the value is read from the class-attribute `__read_sql__`.

```python
# Default select-all
Widget.redpanda(engine)

# Supply a custom query to refine data set
query = sqlalchemy.orm.Query(Widget).filter(Widget.kind=="fizzer")
Widget.redpanda(engine, query)

# Supply **read_sql keyword-args to alter returned DataFrame
Widget.redpanda(engine, query, index_col="id", parse_dates="timestamp")
```


## Parsing DataFrames as Models

The `redparse()` class-method handles the reverse translation of a DataFrame into a collection of SQLAlchemy models.

Use the `parse_index` flag to parse a named index as a model attribute:

```python
frame = pandas.DataFrame({
    datetime.utcnow() : {"name" : "foo", "kind" : "fizzer", "units" : 10 },
    datetime.utcnow() : {"name" : "goo", "kind" : "buzzer", "units" : 11 },
    datetime.utcnow() : {"name" : "hoo", "kind" : "bopper", "units" : 12 }
}).T
frame.index.name = 'timestamp'

widgetgen = Widget.redparse(frame, parse_index=True)
for widget in widgetgen:
    print widget
```
