# Selecting the Python web server

These notes talk about what the available Python web server options are when using the default OpenShift S2I builder for Python, as well as alternative S2I builders for Python.

## Default Python web server options

S2I builders for OpenShift can be magic, but by being magic it means that they can also be quite opinionated. Where a particular programming language has a large range of options for hosting web applications, this can mean that limited options might be allowed for what web server you can run.

For the case of the Python S2I builder, it steers you towards using the ``gunicorn`` WSGI server, that being the only one of the more popular Python WSGI servers that it has specific inbuilt support for.

What dictates what Python web servers can be run and when, is the ``run`` script of the Python S2I builder. The checks it makes in order are as follows:

1. If an ``app.py`` file exists for the application, it is assumed to provide the complete Python web application. It will be run as ``python app.py``.
2. If the ``gunicorn`` module has been installed, a search is made for a ``wsgi.py`` file and if  found ``gunicorn`` will be run, using the ``wsgi.py`` file as a Python module containing the WSGI application entry point. The default ``gunicorn`` configuration of a single process and single threaded ``sync`` worker will be used.
3. If Django is installed and a ``manage.py`` file exists, the Django development server will be run for a Django based application.

Unfortunately all three of these are not necessarily good defaults for various reasons.

An ``app.py`` does allow you to run any Python web server, ASYNC or WSGI, so long as it can be imported as a module and run. The problem is that the Python S2I builder does not provide PID '1' zombie reaping protection built in. If a Python web server doesn't handle reaping of zombie processes, you can run into resource usage problems, or problems with process management.

For further information on the PID '1' zombie reaping problem you can check out:

* [Issues with running as PID 1 in a Docker container](http://blog.dscpl.com.au/2015/12/issues-with-running-as-pid-1-in-docker.html) - http://blog.dscpl.com.au/2015/12/issues-with-running-as-pid-1-in-docker.html

In the case of ``gunicorn``, because no true mutithreaded worker exists, to increase capacity you have to use more processes, or scale up the number of pods. This will result in more memory being used. For primarily I/O bound applications as most web applications generally are, being able to use some level of multithreading allows better use of memory resources. The ``eventlet`` and ``gevent`` workers are not suitable as a general purpose replacement for multithreading. The ``gunicorn`` WSGI server also lacks a way of directly hosting static files.

The fallback to using the Django development server if Django is being used is the most problematic. The Django development server is not suitable for production deployments. It can only handle one request at a time, does not handle the PID '1' zombie reaping problem, as well as having other potential performance issues with the way it handles static files and due to its code reloading mechanism.

One could technically run other command line Python web servers by implementing an ``app.py`` which uses ``os.execl()`` to execute them, but when using a S2I builder you are prevented from installing additional system packages. This is an issue because the default Python S2I builder doesn't install the Apache HTTPD server, which would be required if for example you wanted to run the ``mod_wsgi-express`` server.

## Alternative Python S2I builders

The OpenShift S2I Python builder will no doubt be improved over time, offering a more general purpose solution to support alternate production grade Python web servers, as well as built in protection for the PID '1' zombie reaping problem.

Even so, as the expertise as to what is the best way to setup a Docker container for hosting Python web applications, as well as knowledge of the best server options and configuration likely lies within the wider Python web community, it would be an ideal opportunity for the Python web community to come up with a best of bread S2I builder for Python themselves and drive its development.

One such alternative S2I builder for Python which is available is ``grahamdumpleton/mod-wsgi-docker-s2i``.

* https://hub.docker.com/r/grahamdumpleton/mod-wsgi-docker-s2i/

Although this has ``mod_wsgi`` listed in the name, it is actually a general purpose S2I builder for Python and allows the Python web server used to be overridden. Right now it serves as a proof of concept of what could be done and it is hoped that it could become the model for a community supported, general purpose, S2I builder for Python.

## Using mod_wsgi-express on OpenShift

To use the ``mod_wsgi`` based Python S2I builder with a Django application will require a couple of additions be made to the application code in the Git repository. This is necessary as it doesn't bake in certain behaviour as a default as the philosophy around the ``mod_wsgi`` S2I builder is that users should explicitly enable actions because they know they will need it and not rely on guesses by the S2I builder as to what needs to be done.

The fact that the default Python S2I builder does do certain things as a default can instead causes unexpected problems that may be hard to debug when scaling up a Python web application. A user may not know that they should disable the default behaviour.

An updated version of our sample Django 'Hello World' application designed to work with the ``mod_wsgi`` based Python S2I builder can be found in the Git repository:

* [https://github.com/GrahamDumpleton/django-hello-world-v2](https://github.com/GrahamDumpleton/django-hello-world-v2)

The ``mod_wsgi`` based Python S2I builder provides versions for:

* Python 2.7
* Python 3.3
* Python 3.4
* Python 3.5

To deploy this version of our sample Django 'Hello World' application using Python 3.5, we use the ``oc new-app`` command:

```
$ oc new-app grahamdumpleton/mod-wsgi-docker-s2i:python-3.5~https://github.com/GrahamDumpleton/django-hello-world-v2.git
--> Found Docker image b8dbbcb (2 weeks old) from Docker Hub for "grahamdumpleton/mod-wsgi-docker-s2i:python-3.5"
    * An image stream will be created as "mod-wsgi-docker-s2i:python-3.5" that will track this image
    * A source build using source code from https://github.com/GrahamDumpleton/django-hello-world-v2.git will be created
      * The resulting image will be pushed to image stream "django-hello-world-v2:latest"
      * Every time "mod-wsgi-docker-s2i:python-3.5" changes a new build will be triggered
    * This image will be deployed in deployment config "django-hello-world-v2"
    * Port 80/tcp will be load balanced by service "django-hello-world-v2"
--> Creating resources with label app=django-hello-world-v2 ...
    ImageStream "mod-wsgi-docker-s2i" created
    ImageStream "django-hello-world-v2" created
    BuildConfig "django-hello-world-v2" created
    DeploymentConfig "django-hello-world-v2" created
    Service "django-hello-world-v2" created
--> Success
    Build scheduled for "django-hello-world-v2" - use the logs command to track its progress.
    Run 'oc status' to view your app.
```

We need to once again expose the application to make it publicly available.

```
$ oc expose service django-hello-world-v2
route "django-hello-world-v2" exposed
```

## Specifying the WSGI application

When using the ``mod_wsgi`` based Python S2I builder, you need to explicitly tell it what the WSGI application entry point is. There is no attempt to automatically try and work out what the WSGI application entry point is as this is a potentially error prone process and can lead to incorrect guesses.

If you do not set up the WSGI application entry point, then ``mod_wsgi-express`` will display a default splash page. So you will know the WSGI server is at least working, but it will not be your WSGI application that is running.

To specify the WSGI application entry point for ``mod_wsgi-express`` we need to supply additional options through the ``.whiskey/server_args`` file. By default ``mod_wsgi-express`` expects to be given the path for a WSGI script file. For our Django application, it is structured as a package, so we want to instead provide a module name for the WSGI application entry point. The options we add to the ``.whiskey/server_args`` file is therefore:

```
--application-type module hello_world.wsgi
```

The option ``--application-type module`` indicated that a module name is being supplied. This is followed up by the module name. This corresponds to the file ``hello_world/wsgi.py``.

## Serving up of static file assets

Our Django application has various static file assets. In our 'Hello World' application this will only be what is required for the Django admin interface, but would grow when we start building out the application.

When using the Django builtin development server, such static file assets are handled automatically. When using a separate WSGI server like ``mod_wsgi-express`` we need to configure the Django settings to indicate where static file assets should be placed and then collect them together when building our application for deployment.

To indicate where the static file assets should be placed we add to the Django settings module at ``hello_world/settings.py`` the setting:

```
STATIC_ROOT = os.path.join(BASE_DIR, 'static')
```

When deploying a Django application it is necessary to trigger the collection of static file assets from their various locations into this directory. This is done using the Django management command called ``collectstatic``. To have this be triggered as part of the build phase when the S2I builder is run we add an executable script file ``.whiskey/action_hooks/build`` and place in it:

```
#!/usr/bin/env bash

set -x

python manage.py collectstatic --noinput
```

This script is one of a number of hook scripts supported by the ``mod_wsgi`` based Python S2I builder. The hook scripts when supplied will be triggered appropriately during the build or deployment phases for the application, allowing an application to customise these steps. They mirror the hook scripts from OpenShift 2.

Worth noting is that such hook scripts are not supported by the Python S2I builder of OpenShift 3, even though they were an important feature of OpenShift 2.

Finally, we now need to tell ``mod_wsgi-express`` where the static file assets are located and under what URL they should be hosted. This is doing using the ``--url-alias`` option in the ``.whiskey/server_args`` file.

```
--url-alias /static static
```

The URL path of ``/static`` needs to match what was used for the ``STATIC_URL`` setting in the Django settings module for the application.

In supplying the ``--url-alias`` option, the Apache HTTPD web server started up by ``mod_wsgi-express`` will now host the static files directly. Using the Apache HTTPD web server, rather than relying on a WSGI middleware such as ``WhiteNoise`` will result in better performance.

## Operating behind a proxy router

OpenShift provides a platform for easily scaling up the number of instances of your web application. When this is done, OpenShift automatically handles the load balancing of requests across the multiple instance.

In order to implement this, your web application will always be run behind a proxy router. By default for OpenShift this is ``haproxy``, although it is possible to substitute other routers such as a hardware based F5.

Being behind a proxy router means that you do not see the exact HTTP request headers that the proxy router saw. Instead the request headers will reflect what is the internal address of your web application.

This is important because it means that the details as to what your web application was accessed as, as passed through the WSGI request ``environ`` will be wrong. To accomodate this, it is necessary to consult special request headers added by the proxy router, which give details about the original host, port and scheme that the HTTP request was received on.

To deal with this you would normally need to wrap your WSGI application with a special WSGI middleware that fixed up the WSGI ``environ`` for you. That is, it is not something that would be automatically done. In a PaaS environment it is too easy to forget to do this, plus any WSGI middlemare may not do it correctly.

When using ``mod_wsgi-express`` you can avoid the need to do this yourself as ``mod_wsgi-express`` can do it for you.

To enable this feature we will provide additional options to ``mod_wsgi-express``, telling it what are the trusted proxy headers set by the proxy router. These are added to the ``.whiskey/server_args`` file.

```
--trust-proxy-header X-Forwarded-For --trust-proxy-header X-Forwarded-Port --trust-proxy-header X-Forwarded-Scheme
```

With this in place the WSGI request ``environ`` will now be corrected even before it is passed through to your WSGI application. Importantly, any equivalent request headers often used to indicate this information will be scrubbed from the WSGI request ``environ``. This is to ensure that a HTTP client cannot try and spoof the proxy information where a WSGI middleware was present that looked for more than one possible request header.


