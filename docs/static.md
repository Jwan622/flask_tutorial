# Static files in flask

The authentication views and templates work, but they look very plain right now. Some CSS can be added to add style to the HTML layout you constructed. The style won’t change, so it’s a static file rather than a template.

Flask automatically adds a static view that takes a path relative to the flaskr/static directory and serves it. The base.html template already has a link to the style.css file:


Besides CSS, other types of static files might be files with JavaScript functions, or a logo image. They are all placed under the flaskr/static directory and referenced with url_for('static', filename='...').

This tutorial isn’t focused on how to write CSS, so you can just copy the following into the flaskr/static/style.css file:
