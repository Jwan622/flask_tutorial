# Blueprint

A view function is the code you write to respond to requests to your application. Flask uses patterns to match the incoming request URL to the view that should handle it. The view returns data that Flask turns into an outgoing response. Flask can also go the other direction and generate a URL to a view based on its name and arguments.

A Blueprint is a way to organize a group of related views and other code. Rather than registering views and other code directly with an application, they are registered with a blueprint. Then the blueprint is registered with the application when it is available in the factory function.

Flaskr will have two blueprints, one for authentication functions and one for the blog posts functions. The code for each blueprint will go in a separate module. Since the blog needs to know about authentication, you’ll write the authentication one first.


```python3
import functools

from flask import (
    Blueprint, flash, g, redirect, render_template, request, session, url_for
)
from werkzeug.security import check_password_hash, generate_password_hash

from flaskr.db import get_db

bp = Blueprint('auth', __name__, url_prefix='/auth')
```

- This creates a Blueprint named 'auth'. 
- Like the application object, the blueprint needs to know where it’s defined, so __name__ is passed as the second argument. The `url_prefix` will be prepended to all the URLs associated with the blueprint.

Import and register the blueprint from the factory using `app.register_blueprint()`. `register_blueprint` is a method of the Flask class used to register a Blueprint. A Blueprint is a way to organize a group of related views and other code. It’s akin to a mini-application that is a component of the main application. It helps in making Flask applications more modular and scalable.


```python3
def create_app():
    app = ...
    # existing code omitted

    from . import auth
    app.register_blueprint(auth.bp)

    return app
```

This `bp` object is expected to be a Blueprint instance, which organizes a group of related routes and other view-related code.
In essence, the code is organizing the structure of a Flask application by modularizing the authentication-related routes and functionalities into a separate Blueprint (`auth.bp`), and then registering that Blueprint with the main Flask application instance (app). 

The authentication blueprint will have views to register new users and to log in and log out.

## Auth blueprint and view
When the user visits the `/auth/register URL`, the register view will return HTML with a form for them to fill out. When they submit the form, it will validate their input and either show the form again with an error message or create the new user and go to the login page.

For now you will just write the view code. On the next page, you’ll write templates to generate the HTML form.

```python3
@bp.route('/register', methods=('GET', 'POST'))
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        db = get_db()
        error = None

        if not username:
            error = 'Username is required.'
        elif not password:
            error = 'Password is required.'

        if error is None:
            try:
                db.execute(
                    "INSERT INTO user (username, password) VALUES (?, ?)",
                    (username, generate_password_hash(password)),
                )
                db.commit()
            except db.IntegrityError:
                error = f"User {username} is already registered."
            else:
                return redirect(url_for("auth.login"))

        flash(error)

    return render_template('auth/register.html')
```

Here’s what the register view function is doing:

1. @bp.route associates the URL /register with the register view function. When Flask receives a request to /auth/register, it will call the register view and use the return value as the response.

2. If the user submitted the form, request.method will be 'POST'. In this case, start validating the input.

3. `request.form` is a special type of dict mapping submitted form keys and values. The user will input their username and password.

4. Validate that username and password are not empty.

5. If validation succeeds, insert the new user data into the database.
 - db.execute takes a SQL query with `?` placeholders for any user input, and a tuple of values to replace the placeholders with. The database library will take care of escaping the values so you are not vulnerable to a SQL injection attack.

 - For security, passwords should never be stored in the database directly. Instead, `generate_password_hash()` is used to securely hash the password, and that hash is stored. Since this query modifies data, db.commit() needs to be called afterwards to save the changes.

 - A `sqlite3.IntegrityError` will occur if the username already exists, which should be shown to the user as another validation error.

6. After storing the user, they are redirected to the login page. `url_for()` generates the URL for the login view based on its name. This is preferable to writing the URL directly as it allows you to change the URL later without changing all code that links to it. `redirect()` generates a redirect response to the generated URL.

7. If validation fails, the error is shown to the user. `flash()` stores messages that can be retrieved when rendering the template.

8. When the user initially navigates to `auth/register`, or there was a validation error, an HTML page with the registration form should be shown. `render_template()` will render a template containing the HTML, which you’ll write in the next step of the tutorial.

This view follows the same logic as above but it's for login.

```python3
@bp.route('/login', methods=('GET', 'POST'))
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        db = get_db()
        error = None
        user = db.execute(
            'SELECT * FROM user WHERE username = ?', (username,)
        ).fetchone()

        if user is None:
            error = 'Incorrect username.'
        elif not check_password_hash(user['password'], password):
            error = 'Incorrect password.'

        if error is None:
            session.clear()
            session['user_id'] = user['id']
            return redirect(url_for('index'))

        flash(error)

    return render_template('auth/login.html')
```


There are a few differences from the register view:

1. The user is queried first and stored in a variable for later use.

`fetchone()` returns one row from the query. If the query returned no results, it returns None. Later, fetchall() will be used, which returns a list of all results.

2. `check_password_hash()` hashes the submitted password in the same way as the stored hash and securely compares them. If they match, the password is valid.

3. `session` is a dict that stores data across requests. When validation succeeds, the user’s id is stored in a new session. The data is stored in a cookie that is sent to the browser, and the browser then sends it back with subsequent requests. Flask securely signs the data so that it can’t be tampered with.

Now that the user’s id is stored in the session, it will be available on subsequent requests. At the beginning of each request, if a user is logged in their information should be loaded and made available to other views.

```python3
@bp.before_app_request
def load_logged_in_user():
    user_id = session.get('user_id')

    if user_id is None:
        g.user = None
    else:
        g.user = get_db().execute(
            'SELECT * FROM user WHERE id = ?', (user_id,)
        ).fetchone()
```

`bp.before_app_request()` registers a function that runs before the view function, **no matter what URL is requested**. `load_logged_in_user` checks if a user id is stored in the session and gets that user’s data from the database, storing it on g.user, which lasts for the length of the request. If there is no user id, or if the id doesn’t exist, `g.user` will be None.

### Logout

To log out, you need to remove the user id from the session. Then `load_logged_in_user` won’t load a user on subsequent requests because the first line gets the `user_id` from the session object.


```python3
@bp.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('index'))
```


## Requiring auth on other views

Creating, editing, and deleting blog posts will require a user to be logged in. A decorator can be used to check this for each view it’s applied to.

```python3
def login_required(view):
    @functools.wraps(view)
    def wrapped_view(**kwargs):
        if g.user is None:
            return redirect(url_for('auth.login'))

        return view(**kwargs)

    return wrapped_view
```

- `url_for('auth.login')`is a Flask function that generates the URL for the login view.
- the original view function is called with the provided arguments `(**kwargs)`.

the decorator could be used like this:

```python3
@app.route('/some-private-page')
@login_required
def private_page():
    # Code for the view goes here. This will only be executed if the user is authenticated.
```

## Endpoints and urls

The `url_for()` function generates the URL to a view based on a name and arguments. The name associated with a view is also called the endpoint, and by default it’s the same as the name of the view function.

For example, the `hello()` view that was added to the app factory earlier in the tutorial has the name 'hello' and can be linked to with `url_for('hello')`. If it took an argument, which you’ll see later, it would be linked to using `url_for('hello', who='World')`.

When using a blueprint, the name of the blueprint is prepended to the name of the function, so the endpoint for the login function you wrote above is `'auth.login'` because you added it to the 'auth' blueprint.


## Views

You’ve written the authentication views for your application, but if you’re running the server and try to go to any of the URLs, you’ll see a TemplateNotFound error. That’s because the views are calling `render_template()`, but you haven’t written the templates yet. The template files will be stored in the `templates` directory inside the `flaskr` package.

Templates are files that contain static data as well as placeholders for dynamic data. A template is rendered with specific data to produce a final document. Flask uses the Jinja template library to render templates.

In your application, you will use templates to render HTML which will display in the user’s browser. In Flask, Jinja is configured to **autoescape any data that is rendered in HTML templates. This means that it’s safe to render user input; any characters they’ve entered that could mess with the HTML, such as < and > will be escaped with safe values that look the same in the browser but don’t cause unwanted effects.**

Jinja looks and behaves mostly like Python. Special delimiters are used to distinguish Jinja syntax from the static data in the template. Anything between {{ and }} is an expression that will be output to the final document. {% and %} denotes a control flow statement like if and for. Unlike Python, blocks are denoted by start and end tags rather than indentation since static text within a block could change indentation.

Each page in the application will have the same basic layout around a different body. Instead of writing the entire HTML structure in each template, each template will extend a base template and override specific sections.

```python3
<!doctype html>
<title>{% block title %}{% endblock %} - Flaskr</title>
<link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
<nav>
  <h1>Flaskr</h1>
  <ul>
    {% if g.user %}
      <li><span>{{ g.user['username'] }}</span>
      <li><a href="{{ url_for('auth.logout') }}">Log Out</a>
    {% else %}
      <li><a href="{{ url_for('auth.register') }}">Register</a>
      <li><a href="{{ url_for('auth.login') }}">Log In</a>
    {% endif %}
  </ul>
</nav>
<section class="content">
  <header>
    {% block header %}{% endblock %}
  </header>
  {% for message in get_flashed_messages() %}
    <div class="flash">{{ message }}</div>
  {% endfor %}
  {% block content %}{% endblock %}
</section>
```

`g` is automatically available in templates. Based on if `g.user` is set or not (from `load_logged_in_user`), either the username and a log out link are displayed, OR links to register and log in are displayed. `url_for()` is also automatically available, and is used to generate URLs to views instead of writing them out manually.

After the page title, and before the content, the template loops over each message returned by `get_flashed_messages()`. You used flash() in the views to show error messages, and this is the code that will display them.

There are three blocks defined here that will be overridden in the other templates:

`{% block title %}` will change the title displayed in the browser’s tab and window title. This is in the browser tab. Hover of it and you'll see. You'll see this:
`<title>{% block title %}{% endblock %} - Flaskr</title>`

`{% block header %}` is similar to title but will change the title displayed on the page. The header section simply wraps the title in <h1></h1> tags.

`{% block content %}` is where the content of each page goes, such as the login form or a blog post.

The base template is directly in the `templates` directory. To keep the others organized, the templates for a blueprint will be placed in a directory with the same name as the blueprint.

# Views extending them

`{% extends 'base.html' %}` tells Jinja that this template should replace the blocks from the base template. All the rendered content must appear inside {% block %} tags that override blocks from the base template.

A useful pattern used here is to place `{% block title %}` inside `{% block header %}`. This will set the title block and then output the value of it into the header block, so that both the window and page share the same title without writing it twice.

The input tags are using the required attribute here. This tells the browser not to submit the form until those fields are filled in. If the user is using an older browser that doesn’t support that attribute, or if they are using something besides a browser to make requests, you still want to validate the data in the Flask view. It’s important to always fully validate the data on the server, even if the client does some validation as well.

## Play with the app
Now that the authentication templates are written, you can register a user. Make sure the server is still running (flask run if it’s not), then go to http://127.0.0.1:5000/auth/register.

Try clicking the “Register” button without filling out the form and see that the browser shows an error message. Try removing the required attributes from the register.html template and click “Register” again. Instead of the browser showing an error, the page will reload and the error from flash() in the view will be shown.

Fill out a username and password and you’ll be redirected to the login page. Try entering an incorrect username, or the correct username and incorrect password. If you log in you’ll get an error because there’s no index view to redirect to yet.

