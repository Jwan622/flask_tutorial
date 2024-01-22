# What is flask

A Flask application is an instance of the class Flask. Everything related to the application, such as configuration and URLs, will be registered in this class.

The easiest way to create a Flask application is to create a global instance Flaskdirectly on top of your code, like the “Hello, world!” example did. on the previous page. Although this is simple and useful in some cases, it can cause some complicated problems when the project grows.

Instead of creating an instance of Flask globally, you'll create it inside a function. This feature is known as the application factory . Any configuration, registration, and other settings that the application needs will occur within the function, then the application will be returned.


```python
import os

from flask import Flask


def create_app(test_config=None):
    # create and configure the app
    app = Flask(__name__, instance_relative_config=True)
    app.config.from_mapping(
        SECRET_KEY='dev',
        DATABASE=os.path.join(app.instance_path, 'flaskr.sqlite'),
    )

    if test_config is None:
        # load the instance config, if it exists, when not testing
        app.config.from_pyfile('config.py', silent=True)
    else:
        # load the test config if passed in
        app.config.from_mapping(test_config)

    # ensure the instance folder exists
    try:
        os.makedirs(app.instance_path)
    except OSError:
        pass

    # a simple page that says hello
    @app.route('/hello')
    def hello():
        return 'Hello, World!'

    return app
```

## Explanation of above code

Flask: Flask is a micro web framework for Python. It's widely used for web development due to its simplicity and flexibility.

`__name__`: When you use __name__ in the Flask constructor, it determines the root path for the application. This allows Flask to know where to look for templates, static files, and so on. The __name__ variable in Python represents the name of the current Python script.

`instance_relative_config=True`: This argument tells Flask that configuration files are relative to the instance folder. The instance folder is located outside the actual application package and can hold local data that shouldn't be committed to version control, such as configuration secrets and the local database file. This is especially useful for keeping certain settings or files specific to a particular instance of the application, separate from the main application and its version control.

Creating an instance of Flask: The `app = Flask(__name__, instance_relative_config=True)` line is creating an instance of the Flask class and assigning it to the variable app. This app variable becomes the WSGI (Web Server Gateway Interface) application.

A typical Flask application would use this app object to handle requests and responses. For example, you might define routes and views as functions, using decorators provided by Flask. 

basic example:

```python3
from flask import Flask

# Create an instance of the Flask class
app = Flask(__name__, instance_relative_config=True)

# Define a route for the root URL
@app.route('/')
def index():
    return 'Hello, world!'

# Run the application
if __name__ == '__main__':
    app.run(debug=True)
```


## explanation of db.py file

the code:

```python3
import sqlite3

import click
from flask import current_app, g


def get_db():
    if 'db' not in g:
        g.db = sqlite3.connect(
            current_app.config['DATABASE'],
            detect_types=sqlite3.PARSE_DECLTYPES
        )
        g.db.row_factory = sqlite3.Row

    return g.db


def close_db(e=None):
    db = g.pop('db', None)

    if db is not None:
        db.close()
```

`g` is a special object that is unique for each request. It is used to store data that might be accessed by multiple functions during the request. The connection is stored and reused instead of creating a new connection if get_db is called a second time in the same request.

`import sqlite3`: This imports the SQLite3 module, which is a lightweight disk-based database that doesn't require a separate server process. It’s often used for small to medium-sized applications, prototyping, and development.

```
import click and from flask import current_app, g:
```

`click` is a Python package for creating command-line interfaces.
`current_app` is a proxy object in Flask that points to the application handling the current activity. It's dynamically mapped to the application that is handling the current request. This is useful in larger applications, where you might have multiple applications or blueprints.
`g` is a global namespace object provided by Flask. It's a simple namespace object that can store data during an application context. Data stored in g persists for the length of a request. It's often used for storing database connections and other data that needs to be accessible throughout the handling of a request.

`get_db()` function:
This function is responsible for setting up a connection to the SQLite database.
It checks if a database connection (db) already exists in the g object. If not, it creates a new database connection using sqlite3.connect(). The database file path is retrieved from the Flask application's config object `(current_app.config['DATABASE'])`.
The `detect_types=sqlite3.PARSE_DECLTYPES` argument enables type detection, allowing SQLite to return Python types.
The `row_factory` attribute is set to sqlite3.Row to access columns by name.
Finally, it returns the database connection object.
`close_db(e=None)` function:

This function is responsible for closing the database connection.
It tries to pop the db connection from the `g` object. If a connection is found `(if db is not None)`, it closes the connection.

In summary, this code manages database connections for each request in a Flask application. It ensures that the database connection is opened at the beginning of a request and closed when the request ends, using Flask's application context global objects g and current_app.


```python3
def init_app(app):
    app.teardown_appcontext(close_db)
    app.cli.add_command(init_db_command)
```

`app.teardown_appcontext()` tells Flask to call that function when cleaning up after returning the response.
`app.cli.add_command()` adds a new command that can be called with the flask command.

click.command() defines a command line command called init-db that calls the init_db function and shows a success message to the user.

Now that `init-db` has been registered with the app, it can be called using the flask command, similar to the run command from the previous page.

