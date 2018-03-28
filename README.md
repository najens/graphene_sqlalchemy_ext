# graphene_sqlalchemy_ext
graphene_sqlalchemy_ext is an extension to graphene_sqlalchemy with built-in sort functionality for your GraphQL queries.

## Installation

For instaling graphene, just run this command in your shell

```bash
pip install git+git://github.com/najens/graphene_sqlalchemy_ext.git
```

## Examples

Here is a simple SQLAlchemy model:

```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy

# Initialize app
app = Flask(__name__)

# Initialize Flask_SQLAlchemy
db = SQLAlchemy(app)

# Define User model
class UserModel(db.Model):
    __tablename__ = 'user'
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(32))
    last_name = db.Column(db.String(32))
```

To create a GraphQL schema for it you simply have to write the following:

```python
from graphene_sqlalchemy import SQLAlchemyObjectType, SQLAlchemyConnectionField

class User(SQLAlchemyObjectType):
    class Meta:
        model = UserModel
        interfaces = (relay.Node, )

class Query(graphene.ObjectType):
    node = relay.Node.Field()
    user = relay.Node.Field(User)
    users = SQLAlchemyConnectionField(User)

schema = graphene.Schema(query=Query)
```

Then you can simply query the schema to get all users:

```python
query = '''
    query {
      users {
        node {
          edges {
            name,
            lastName
          }
        }
      }
    }
'''

Get a single user by id:

query = '''
    query {
      user(id: 1) {
        edges {
          node {
            name,
            lastName
          }
        }
      }
    }
'''

And sort the users:

query = '''
    query {
      users(sort: name_desc) {
        node {
          edges {
            name,
            lastName
          }
        }
      }
    }
'''

result = schema.execute(query, context_value={'session': db.session})
```

You may also subclass SQLAlchemyConnectionField to add filters to your queries
by overriding the get_query method

```python
from graphene_sqlalchemy import SQLAlchemyObjectType, SQLAlchemyConnectionField
from graphene_sqlalchemy.utils import get_query

class MySQLAlchemyConnectionField(SQLAlchemyConnectionField):

    @classmethod
    def get_query(cls, model, info, sort=None, **args):
        RELAY_ARGS = ['first', 'last', 'before', 'after']
        query = get_query(model, info.context)
        for field, values in args.items():
            if field not in RELAY_ARGS:
            query = query.filter(db.or_(getattr(model, field) == item for item in values))
        if sort is not None:
            if isinstance(sort, str):
                query = query.order_by(sort.order)
            else:
                query = query.order_by(*(value.order for value in sort))
        return query

class User(SQLAlchemyObjectType):
    class Meta:
        model = UserModel
        interfaces = (relay.Node, )

class Query(graphene.ObjectType):
    node = relay.Node.Field()
    user = relay.Node.Field(User)
    users = MySQLAlchemyConnectionField(User,
            name=graphene.List(graphene.String),
            lastName=graphene.List(graphene.String))

schema = graphene.Schema(query=Query)

Then you can simply query the schema to filter users:

```python
query = '''
    query {
      users(name: "Michael", lastName: "Jackson") {
        node {
          edges {
            name,
            lastName
          }
        }
      }
    }
'''

result = schema.execute(query, context_value={'session': db.session})
```

You may also subclass SQLAlchemyObjectType by providing `abstract = True` in
your subclasses Meta:
```python
from graphene_sqlalchemy import SQLAlchemyObjectType

class ActiveSQLAlchemyObjectType(SQLAlchemyObjectType):
    class Meta:
        abstract = True

    @classmethod
    def get_node(cls, info, id):
        return cls.get_query(info).filter(
            and_(cls._meta.model.deleted_at==None,
                 cls._meta.model.id==id)
            ).first()

class User(ActiveSQLAlchemyObjectType):
    class Meta:
        model = UserModel

class Query(graphene.ObjectType):
    users = graphene.List(User)

    def resolve_users(self, info):
        query = User.get_query(info)  # SQLAlchemy query
        return query.all()

schema = graphene.Schema(query=Query)
```
