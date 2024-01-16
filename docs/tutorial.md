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
