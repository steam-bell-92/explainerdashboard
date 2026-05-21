Deployment
**********

When deploying your dashboard it is better not to use the built-in flask
development server but use a more robust production server like ``gunicorn`` or ``waitress``.
Probably `gunicorn <https://gunicorn.org/>`_ is a bit more fully featured and
faster but only works on unix/linux/osx, whereas
`waitress <https://docs.pylonsproject.org/projects/waitress/en/stable/>`_ also works
on Windows and has very minimal dependencies.

Install with either ``pip install gunicorn`` or ``pip install waitress``.

Storing explainer and running default dashboard with gunicorn
=============================================================

Before you start a dashboard with gunicorn you need to store both the explainer
instance and and a configuration for the dashboard::

    from explainerdashboard import ClassifierExplainer, ExplainerDashboard

    explainer = ClassifierExplainer(model, X, y)
    db = ExplainerDashboard(explainer, title="Cool Title", shap_interaction=False)
    db.to_yaml("dashboard.yaml", explainerfile="explainer.joblib", dump_explainer=True)

Now you re-load your dashboard and expose a flask server as ``app`` in ``dashboard.py``::

    from explainerdashboard import ExplainerDashboard

    db = ExplainerDashboard.from_config("dashboard.yaml")
    app = db.flask_server()


.. highlight:: bash

If you named the file above ``dashboard.py``, you can now start the gunicorn server with::

    $ gunicorn dashboard:app

If you want to run the server server with for example three workers, binding to
port ``8050`` you launch gunicorn with::

    $ gunicorn -w 3 -b localhost:8050 dashboard:app

If you now point your browser to ``http://localhost:8050`` you should see your dashboard.
Next step is finding a nice url in your organization's domain, and forwarding it
to your dashboard server.

With waitress you would call::

    $ waitress-serve --port=8050 dashboard:app

.. highlight:: python

Although you can all use the ``waitress`` directly from the dashboard by passing
the ``use_waitress=True`` flag to ``.run()``::

    ExplainerDashboard(explainer).run(use_waitress=True)


Deploying dashboard as part of Flask app on specific route
==========================================================

.. highlight:: python

Another way to deploy the dashboard is to first start a ``Flask`` app, and then
use this app as the backend of the Dashboard, and host the dashboard on a specific
route. This way you can for example host multiple dashboard under different urls.
You need to pass the Flask ``server`` instance and the ``url_base_pathname`` to the
``ExplainerDashboard`` constructor, and then the dashboard itself can be found
under ``db.app.index``::

    from flask import Flask

    app = Flask(__name__)

    [...]

    db = ExplainerDashboard(explainer, server=app, url_base_pathname="/dashboard/")

    @app.route('/dashboard')
    def return_dashboard():
        return db.app.index()


.. highlight:: bash

Now you can start the dashboard by::

    $ gunicorn -b localhost:8050 dashboard:app

And you can visit the dashboard on ``http://localhost:8050/dashboard``.


Deploying to heroku
===================

In case you would like to deploy to `heroku <www.heroku.com>`_ (which is normally
the simplest option for dash apps, see
`dash instructions here <https://dash.plotly.com/deployment>`_). The demonstration
dashboard is hosted on Fly.io at `titanicexplainer.fly.dev <https://titanicexplainer.fly.dev>`_
and on Hugging Face Spaces at
`huggingface.co/spaces/oegedijk/explainingtitanic <https://huggingface.co/spaces/oegedijk/explainingtitanic>`_.

In order to deploy the heroku there are a few things to keep in mind. First of
all you need to add ``explainerdashboard`` and ``gunicorn`` to
``requirements.txt`` (pinning is recommended to force a new build of your environment
whenever you upgrade versions)::

    explainerdashboard==0.3.1
    gunicorn

Select a python runtime compatible with the version that you used to pickle
your explainer in ``runtime.txt``::

    python-3.8.6

(supported versions as of this writing are ``python-3.9.0``, ``python-3.8.6``,
``python-3.7.9`` and ``python-3.6.12``, but check the
`heroku documentation <https://devcenter.heroku.com/articles/python-support#supported-runtimes>`_
for the latest)


And you need to tell heroku how to start your server in ``Procfile``::

    web: gunicorn dashboard:app


Graphviz buildpack
------------------

If you want to visualize individual trees inside your ``RandomForest``, ``xgboost`` or ``lightgbm``
model using the ``dtreeviz`` package you will
need to make sure that ``graphviz`` is installed on your ``heroku`` dyno by
adding the following buildstack (as well as the ``python`` buildpack):
``https://github.com/weibeld/heroku-buildpack-graphviz.git``

(you can add buildpacks through the "settings" page of your heroku project)

Deploying to Fly.io
===================

`Fly.io <https://fly.io>`_ is a good option for running ``explainerdashboard`` as
a small containerized web app.

At minimum you need:

1. A ``dashboard.py`` with a WSGI app object::

    from explainerdashboard import ExplainerDashboard

    db = ExplainerDashboard.from_config("dashboard.yaml")
    app = db.flask_server()

2. A ``requirements.txt`` that includes at least::

    explainerdashboard
    gunicorn

3. A ``Procfile`` or start command equivalent::

    web: gunicorn dashboard:app --bind 0.0.0.0:${PORT:-8080}

Then deploy with ``flyctl``:

.. highlight:: bash

::

    $ fly auth login
    $ fly launch
    $ fly deploy

When prompted during ``fly launch``, set the internal port to ``8080`` if needed
so it matches your ``gunicorn`` bind port.

If your app serves under a subpath (for example behind a proxy), also set
``url_base_pathname``, ``routes_pathname_prefix`` and ``requests_pathname_prefix``
on ``ExplainerDashboard``.

.. highlight:: python

Deploying behind reverse proxies and path prefixes
==================================================

For any platform behind a reverse proxy or ingress (Azure, Kubernetes, AWS ALB, etc),
the key requirement is that Dash base paths and proxy routing agree.

Use a shared base path (for example ``/dashboard/``) for all three settings::

    db = ExplainerDashboard(
        explainer,
        url_base_pathname="/dashboard/",
        routes_pathname_prefix="/dashboard/",
        requests_pathname_prefix="/dashboard/",
    )
    app = db.flask_server()

Quick checks:

1. The page is served from the same base path as Dash API calls.
2. ``/_dash-layout`` and ``/_dash-dependencies`` return ``200`` (not ``404``).
3. ``/_dash-component-suites/*`` assets are reachable.
4. The app binds to ``0.0.0.0:$PORT`` in production.

Deploying to Google Cloud Run
=============================

Cloud Run works well with ``gunicorn`` and a containerized app.

**dashboard.py**::

    from explainerdashboard import ExplainerDashboard

    db = ExplainerDashboard.from_config("dashboard.yaml")
    app = db.flask_server()

**Dockerfile**::

    FROM python:3.11-slim

    WORKDIR /app
    COPY requirements.txt .
    RUN pip install --no-cache-dir -r requirements.txt
    COPY . .

    CMD ["sh", "-c", "gunicorn dashboard:app --bind 0.0.0.0:${PORT}"]

Notes:

1. Keep startup fast (load prebuilt explainer, do not retrain on request).
2. If served under a path prefix, set the three Dash pathname prefixes as above.

Deploying to Kubernetes (Ingress)
=================================

For Kubernetes with ingress path routing, avoid path rewrites unless your Dash prefixes
match the rewritten path exactly.

Example deployment env var:

.. code-block:: yaml

    env:
      - name: APP_BASE_PATH
        value: /dashboard/

Example app setup:

.. highlight:: python

::

    import os
    from explainerdashboard import ExplainerDashboard

    base_path = os.getenv("APP_BASE_PATH", "/")
    db = ExplainerDashboard(
        explainer,
        url_base_pathname=base_path,
        routes_pathname_prefix=base_path,
        requests_pathname_prefix=base_path,
    )
    app = db.flask_server()

If you see ``Loading...`` with ``404`` responses on ``/_dash-*``, fix ingress
path rules and Dash prefixes first.

.. highlight:: python

Deploying in Databricks notebooks
=================================

For model exploration in Databricks, run the dashboard in notebook context and make
sure all Dash pathname prefixes match the Databricks proxy path.

Example::

    from explainerdashboard import ExplainerDashboard

    port = 8050
    # Replace this with your workspace-specific Databricks proxy base path:
    base_path = f"/driver-proxy/o/<org-id>/<cluster-id>/{port}/"

    db = ExplainerDashboard(
        explainer,
        mode="external",
        url_base_pathname=base_path,
        routes_pathname_prefix=base_path,
        requests_pathname_prefix=base_path,
    )
    db.run(port=port)

If you get ``Loading...``, check network requests for ``/_dash-layout`` and
``/_dash-dependencies``; ``404`` usually means proxy prefix mismatch.

Deploying in Kaggle notebooks
=============================

Kaggle is best suited for interactive exploration in notebook sessions, not durable
hosting of a long-running web app.

Recommended pattern:

1. Run in notebook mode (``mode="inline"`` or ``mode="external"``).
2. Keep it lightweight for notebook resources (for example lower ``plot_sample``,
   disable expensive tabs such as ``shap_interaction``).
3. Share results with ``db.save_html("dashboard.html")`` when you need a portable artifact.

Deploying to Hugging Face Spaces
================================

The easiest way to deploy to `Hugging Face Spaces <https://huggingface.co/spaces>`_
is as a Docker Space.

Create a new Space with:

- SDK: ``Docker``
- Visibility: your choice

Add the following files to your Space repository.

**requirements.txt**::

    explainerdashboard
    gunicorn

**dashboard.py**::

    from explainerdashboard import ExplainerDashboard

    db = ExplainerDashboard.from_config("dashboard.yaml")
    app = db.flask_server()

.. highlight:: docker

**Dockerfile**::

    FROM python:3.11-slim

    WORKDIR /app
    COPY requirements.txt .
    RUN pip install --no-cache-dir -r requirements.txt
    COPY . .

    EXPOSE 7860
    CMD ["gunicorn", "dashboard:app", "--bind", "0.0.0.0:7860"]

.. highlight:: python

Push to the Space repository and Hugging Face will build and start the app.
If your dashboard uses tree visualization (``dtreeviz``), make sure system
``graphviz`` is installed in the Docker image.

Docker deployment
=================
.. highlight:: python

You can also deploy a dashboard using docker. You can build the dashboard and store
it inside the container to make sure it is compatible with the container environment.
E.g. **generate_dashboard.py**::

    from sklearn.ensemble import RandomForestClassifier

    from explainerdashboard import *
    from explainerdashboard.datasets import *

    X_train, y_train, X_test, y_test = titanic_survive()
    model = RandomForestClassifier(n_estimators=50, max_depth=5).fit(X_train, y_train)

    explainer = ClassifierExplainer(model, X_test, y_test,
                                    cats=["Sex", 'Deck', 'Embarked'],
                                    labels=['Not Survived', 'Survived'],
                                    descriptions=feature_descriptions)

    # For sklearn/imblearn pipeline models you can alternatively use:
    # explainer = ClassifierExplainer(
    #     pipeline_model, X_test, y_test,
    #     strip_pipeline_prefix=True,
    #     auto_detect_pipeline_cats=True)

    db = ExplainerDashboard(explainer)
    db.to_yaml("dashboard.yaml", explainerfile="explainer.joblib", dump_explainer=True)

**run_dashboard.py**::

    from explainerdashboard import ExplainerDashboard

    db = ExplainerDashboard.from_config("dashboard.yaml")
    db.run(host='0.0.0.0', port=9050, use_waitress=True)

.. highlight:: docker

**Dockerfile**::

    FROM python:3.8

    RUN pip install explainerdashboard

    COPY generate_dashboard.py ./
    COPY run_dashboard.py ./

    RUN python generate_dashboard.py

    EXPOSE 9050
    CMD ["python", "./run_dashboard.py"]

.. highlight:: bash

And build and run the container exposing port ``9050``::

    $ docker build -t explainerdashboard .
    $ docker run -p 9050:9050 explainerdashboard

Reducing memory usage
=====================

If you deploy the dashboard with a large dataset with a large number of rows (``n``)
and a large number of columns (``m``),
it can use up quite a bit of memory: the dataset itself, shap values,
shap interaction values and any other calculated properties are alle kept in
memory in order to make the dashboard responsive. You can check the (approximate)
memory usage with ``explainer.memory_usage()``. In order to reduce the memory
footprint there are a number of things you can do:

1. Not including shap interaction tab.
    Shap interaction values are shape ``n*m*m``, so can take a subtantial amount
    of memory, especially if you have a significant amount of columns ``m``.
2. Setting a lower precision.
    By default shap values are stored as ``'float64'``,
    but you can store them as ``'float32'`` instead and save half the space:
    ```ClassifierExplainer(model, X_test, y_test, precision='float32')```. You
    can also set a lower precision on your ``X_test`` dataset yourself ofcourse.
3. Drop non-positive class shap values.
    For multi class classifiers, by default ``ClassifierExplainer`` calculates
    shap values for all classes. If you are only interested in a single class
    you can drop the other shap values with ``explainer.keep_shap_pos_label_only(pos_label)``
4. Storing row data externally and loading on the fly.
    You can for example only store a subset of ``10.000`` rows in
    the ``explainer`` itself (enough to generate representative importance and dependence plots),
    and store the rest of your millions of rows of input data in an external file
    or database that get loaded one by one with the following functions:

    - with ``explainer.set_X_row_func()`` you can set a function that takes
      an `index` as argument and returns a single row dataframe with model
      compatible input data for that index. This function can include a query
      to a database or fileread.
    - with ``explainer.set_y_func()`` you can set a function that takes
      and `index` as argument and returns the observed outcome ``y`` for
      that index.
    - with ``explainer.set_index_list_func()`` you can set a function
      that returns a list of available indexes that can be queried.

    If the number of indexes is too long to fit in a dropdown you can pass
    ``index_dropdown=False`` which turns the dropdowns into free text fields.
    Instead of an ``index_list_func`` you can also set an
    ``explainer.set_index_check_func(func)`` which should return a bool whether
    the ``index`` exists or not.

    Important: these function can be called multiple times by multiple independent
    components, so probably best to implement some kind of caching functionality.
    The functions you pass can be also methods, so you have access to all of the
    internals of the explainer.


Setting logins and password
===========================

``ExplainerDashboard`` supports `dash basic auth functionality <https://dash.plotly.com/authentication>`_.
``ExplainerHub`` uses ``flask_simple_login`` for its user authentication.

You can simply add a list of logins to the ``ExplainerDashboard`` to force a login
and prevent random users from accessing the details of your model dashboard::

    ExplainerDashboard(explainer, logins=[['login1', 'password1'], ['login2', 'password2']]).run()

Whereas :ref:`ExplainerHub<ExplainerHub>` has somewhat more intricate user management
using ``FlaskLogin``, but the basic syntax is the same. See the
:ref:`ExplainerHub documetation<ExplainerHub>` for more details::

    hub = ExplainerHub([db1, db2], logins=[['login1', 'password1'], ['login2', 'password2']])

Make sure not to check these login/password pairs into version control though,
but store them somewhere safe! ``ExplainerHub`` stores passwords into a hashed
format by default.


Automatically restart gunicorn server upon changes
==================================================

We can use the ``explainerdashboard`` CLI tools to automatically rebuild our
explainer whenever there is a change to the underlying
model, dataset or explainer configuration. And we we can use ``kill -HUP gunicorn.pid``
to force the gunicorn to restart and reload whenever a new ``explainer.joblib``
is generated or the dashboard configuration ``dashboard.yaml`` changes. These two
processes together ensure that the dashboard automatically updates whenever there
are underlying changes.

First we store the explainer config in ``explainer.yaml`` and the dashboard
config in ``dashboard.yaml``. We also indicate which modelfiles and datafiles the
explainer depends on, and which columns in the datafile should be used as
a target and which as index::

    explainer = ClassifierExplainer(model, X, y, labels=['Not Survived', 'Survived'])
    explainer.dump("explainer.joblib")
    explainer.to_yaml("explainer.yaml",
                    modelfile="model.pkl",
                    datafile="data.csv",
                    index_col="Name",
                    target_col="Survival",
                    explainerfile="explainer.joblib",
                    dashboard_yaml="dashboard.yaml")

    db = ExplainerDashboard(explainer, [ShapDependenceTab, ImportancesTab], title="Custom Title")
    db.to_yaml("dashboard.yaml", explainerfile="explainer.joblib")

The ``dashboard.py`` is the same as before and simply loads an ``ExplainerDashboard``
directly from the config file::

    from explainerdashboard import ExplainerDashboard

    db = ExplainerDashboard.from_config("dashboard.yaml")
    app = db.app.server

.. highlight:: bash

Now we would like to rebuild the ``explainer.joblib`` file whenever there is a
change to ``model.pkl``, ``data.csv`` or ``explainer.yaml`` by running
``explainerdashboard build``. And we restart the ``gunicorn`` server whenever
there is a change in ``explainer.joblib`` or ``dashboard.yaml`` by killing
the gunicorn server with ``kill -HUP pid`` To do that we need to install
the python package ``watchdog`` (``pip install watchdog[watchmedo]``). This
package can keep track of filechanges and execute shell-scripts upon file changes.

So we can start the gunicorn server and the two watchdog filechange trackers
from a shell script ``start_server.sh``::

    trap "kill 0" EXIT  # ensures that all three process are killed upon exit

    source venv/bin/activate # activate virtual environment first

    gunicorn --pid gunicorn.pid gunicorn_dashboard:app &
    watchmedo shell-command  -p "./model.pkl;./data.csv;./explainer.yaml" -c "explainerdashboard build explainer.yaml" &
    watchmedo shell-command -p "./explainer.joblib;./dashboard.yaml" -c 'kill -HUP $(cat gunicorn.pid)' &

    wait # wait till user hits ctrl-c to exit and kill all three processes

Now we can simply run ``chmod +x start_server.sh`` and ``./start_server.sh`` to
get our server up and running.

Whenever we now make a change to either one of the source files
(``model.pkl``, ``data.csv`` or ``explainer.yaml``), this produces a fresh
``explainer.joblib``. And whenever there is a change to either ``explainer.joblib``
or ``dashboard.yaml`` gunicorns restarts and rebuild the dashboard.

So you can keep an explainerdashboard running without interuption and simply
an updated ``model.pkl`` or a fresh dataset ``data.csv`` into the directory and
the dashboard will automatically update.
