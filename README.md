### Django-pool
MySQL Connection Pooling with Django(>=2.0.5) and SQLAlchemy(>=1.2.7).

### Why
If `CONN_MAX_AGE` not set in you settings, Django will establish a new MySQL connection for each request and close it after the request.

So if where are too many connections to be handled, it will establish too many connections until running out of MySQL `max_connections`.

Then, MySQL will return the error: `ERROR 1040 (08004): Too many connections`.

It's necessary to maintain a connection pool for a robust Django project.

### Principle
[Reference: MySQL Connection Pooling with Django and SQLAlchemy](http://menendez.com/blog/mysql-connection-pooling-django-and-sqlalchemy/)

Make MySQLdb Database `get_new_connection` from connection pool with SQLAlchemy QueuePool.

So all we need to custom a Database managed by connection pool and get new connection from the pool.

```
pip install mysqlclient
pip install SQLAlchemy
```

**core code**:

```
import MySQLdb as Database
import sqlalchemy.pool as pool
from sqlalchemy.pool import QueuePool
from django.db.backends.mysql.base import DatabaseWrapper as _DatabaseWrapper


Database = pool.manage(Database, poolclass=QueuePool, **settings.SQLALCHEMY_QUEUEPOOL)

class DatabaseWrapper(_DatabaseWrapper):
    def get_new_connection(self, conn_params):
        # return a mysql connection
        conn_params = settings.DATABASES['default']
        return Database.connect(
            host=conn_params['HOST'],
            port=conn_params['PORT'],
            user=conn_params['USER'],
            db=conn_params['NAME'],
            passwd=conn_params['PASSWORD'],
            use_unicode=True,
            charset='utf8',
        )
```

### Usage
settings.py

```
# http://docs.sqlalchemy.org/en/latest/core/pooling.html#sqlalchemy.pool.QueuePool
# http://docs.sqlalchemy.org/en/latest/core/pooling.html#sqlalchemy.pool.Pool.params
SQLALCHEMY_QUEUEPOOL = {
    'pool_size': 10,
    'max_overflow': 10,
    'timeout': 5,
    'recycle': 119,
    'echo': False,
}

DATABASES = {
    'default': {
        #'ENGINE': 'django.db.backends.mysql',
        'ENGINE': 'django_pool',  # can import_module() in your Python path
        'HOST': '127.0.0.1',
        'NAME': 'xxx',
        'USER': 'xxx',
        'PASSWORD': 'xxx',
        'PORT': 3306,
    },
}
```


### Test
If `python manage.py runserver` is OK, now you can use load testing utility [Siege](https://www.joedog.org/siege-home/) to test.

```
siege -r 2 -c 1000 -d 0 http://xxx  # siege --help
```

You can compare the MySQL process list count to the before, and it should work.

Btw, you should get a `SQLAlchemy.exc.Timeout` exception if it does not get a connection during the request.