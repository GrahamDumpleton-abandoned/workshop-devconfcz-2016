# Configuration and application scaling

These notes talk about manually configuring ``mod_wsgi-express`` for deployment of a Python web application.

## Overriding the type of server

When automatic mode of the ``warpdrive`` based Python S2I builder is used, a best guess setup will be applied to get your Python web application running. If you wish to ensure that the correct type of setup is used, the server type can be overridden within the application code by adding the file ``.warpdrive/server_type``.

The server type can be explicitly set to:

* ``shell`` - Will execute the shell script ``app.sh``.
* ``python`` - Will execute the Python script ``app.py``.
* ``paste`` - Will host the WSGI application described by the ``paste.ini`` file.
* ``django`` - Will host the Django application managed by the ``manage.py`` file.

In the cases of ``paste`` and ``django`` server type, ``mod_wsgi-express`` will be used with appropriate options automatically passed to it to make it work.

If you want to ignore the automatically generated configuration for ``paste`` and ``django`` and take full control of the setup of the WSGI server for these, or even have a custom WSGI application, you can set the server type to any of:

* ``gunicorn``
* ``mod_wsgi``
* ``waitress``
* ``uwsgi``

The startup script in ``warpdrive`` will start the selected WSGI server up for you, ensuring that the minimal options are set which will allow it to run inside of the container properly, and listening on the correct port for HTTP requests.

Any options that you then wish to use to further configure the selected WSGI server, would  be added to the ``.warpdrive/server_args`` file.

The structure of ``warpdrive`` is such that support for other WSGI servers could easily be added, alternatively you provide an ``app.sh`` file with launches any other Python WSGI, ASYNC, general purpose web server, or even a custom application that you want to run.

Example Git repositories with the server specific ``server_args`` file can be found at:

* [django-hello-world-v2](https://github.com/GrahamDumpleton/django-hello-world-v2) - server_type = django
* [django-hello-world-v3](https://github.com/GrahamDumpleton/django-hello-world-v3) - server_type = gunicorn
* [django-hello-world-v4](https://github.com/GrahamDumpleton/django-hello-world-v4) - server_type = waitress
* [django-hello-world-v5](https://github.com/GrahamDumpleton/django-hello-world-v5) - server_type = mod_wsgi
* [django-hello-world-v6](https://github.com/GrahamDumpleton/django-hello-world-v6) - server_type = uwsgi

You can use ``oc new-app`` with any of these and the ``warpdrive`` based S2I Python builder. For example, if you wanted to evaluate the Waitress WSGI server use:

```
oc new-app warp0-debian8-python:2.7~https://github.com/GrahamDumpleton/django-hello-world-v4
```

## Increasing application capacity 

When using one of the options above which uses ``mod_wsgi-express``, a reasonably safe configuration which is generally suitable for most Python web applications will be used. It is possible that some configuration settings will need to be overridden, the most common being the need to override the number of processes and threads that ``mod_wsgi-express`` runs with.

The default configuration is a single process with five threads. This may need to be changed to adjust the capacity of the WSGI server. Exactly how it may need to be changed will depend on throughput, response times and whether your code is CPU bound vs I/O bound.

Ideally you should not have to change the code of the application and rebuild an image to adjust WSGI server configuration parameters such as this. These settings can therefore be adjusted using environment variables, meaning that only a redeployment of the existing image is needed.

If you have a I/O bound application with many long running requests and need to increase the capacity for concurrent requests, you can simply change the environment variables for the deployment configuration, increasing both the number of processes and number of threads as felt necessary.

```
$ oc env deploymentconfig django-hello-world-v1 MOD_WSGI_PROCESSES=3 MOD_WSGI_THREADS=15
deploymentconfig "django-hello-world-v1" updated

$ oc env deploymentconfig django-hello-world-v1 --list
# deploymentconfigs django-hello-world-v1, container django-hello-world-v1
MOD_WSGI_PROCESSES=3
MOD_WSGI_THREADS=15
```

Increasing the capacity or adjusting the mix of processes vs threads would be done to try optimise as much as possible the amount of the CPU and memory resources allocated to a pod. This would be done before scaling out the actual number of pods. The number of pod replicas as the second step could be done using.

```
$ oc scale deploymentconfig django-hello-world-v1 --replicas 3
deploymentconfig "django-hello-world-v1" scaled

$ oc get pods --selector app=django-hello-world-v1
NAME                             READY     STATUS    RESTARTS   AGE
django-hello-world-v1-11-f51li   1/1       Running   0          1m
django-hello-world-v1-11-khmod   1/1       Running   0          4m
django-hello-world-v1-11-zqeas   1/1       Running   0          1m
```

## Application source code reloading

Previously when using the default Python S2I builder, the fact that it ran the Django development server meant that application source code would be automatically reloaded if changed. This allowed source code in the container to be changed live by direct editing or using ``oc rsync``.

Even though we are using ``mod_wsgi-express``, automatic source code reloading in the style of the Django development server, can be optionally enabled. This also can be done using an environment variable setting. As before, when making live code changes, you should only be running with a single pod so you know where to change the code.

```
$ oc scale deploymentconfig django-hello-world-v1 --replicas 1

$ oc env deploymentconfig django-hello-world-v1 MOD_WSGI_RELOAD_ON_CHANGES=1
deploymentconfig "django-hello-world-v1" updated

$ oc get pods --selector app=django-hello-world-v1
NAME                            READY     STATUS    RESTARTS   AGE
django-hello-world-v1-2-ek830   1/1       Running   0          32s

$ oc rsync hello_world/ django-hello-world-v1-2-ek830:/app/hello_world/
building file list ... done
./
__init__.py
__init__.pyc
settings.py
settings.pyc
urls.py
urls.pyc
views.py
views.pyc
wsgi.py
wsgi.pyc

sent 5756 bytes  received 336 bytes  812.27 bytes/sec
total size is 9621  speedup is 1.58
```

Remember to never use full application source code reloading on a production system as it will impact performance.

## Specifying the WSGI application

If using the ``django`` server type, although ``mod_wsgi-express`` is used, the primary configuration for Django is still automatically generated. If you had instead explicitly selected ``mod_wsgi``, you will need to provide all the configuration.

To specify the WSGI application entry point for ``mod_wsgi-express`` we need to supply additional options through the ``.warpdrive/server_args`` file. By default ``mod_wsgi-express`` expects to be given the path for a WSGI script file. For our Django application, it is structured as a package, so we want to instead provide a module name for the WSGI application entry point. The options we add to the ``.warpdrive/server_args`` file is therefore:

```
--application-type module --entry-point hello_world.wsgi
```

The option ``--application-type module`` indicated that a module name is being supplied. This is followed up by the module name using the ``--entry-point`` option.

## Serving up of static file assets

Our Django application has various static file assets. In our Django 'Hello World' application this will only be what is required for the Django admin interface, but would grow when we start building out the application.

To indicate where the static file assets should be placed we would already have needed to add to the Django settings module at ``hello_world/settings.py`` the setting:

```
STATIC_ROOT = os.path.join(BASE_DIR, 'static')
```

This is the case even if relying on ``auto`` mode for ``server_type``.

When deploying a Django application it is also necessary to trigger the collection of static file assets from their various locations into this directory. This is done using the Django management command called ``collectstatic``. If using ``auto`` or ``django`` this would be automatically done as part of the build of the Docker image. Because we have indicated that we want to configure everything ourselves with the ``mod_wsgi`` server type, we will need to handle this ourselves. We will get to how that is done later.

Assuming our static files will be in the ``static`` directory, we can tell ``mod_wsgi-express`` where they are and under what URL they should be hosted by using the ``--url-alias`` option in the ``.warpdrive/server_args`` file.

```
--url-alias /static/ static/
```

The URL path of ``/static/`` needs to match what was used for the ``STATIC_URL`` setting in the Django settings module for the application.

In supplying the ``--url-alias`` option, the Apache HTTPD web server started up by ``mod_wsgi-express`` will now host the static files directly. Using the Apache HTTPD web server, rather than relying on a WSGI middleware such as ``WhiteNoise`` will result in better performance. If using ``gunicorn`` or ``waitress`` you would need to install and use WhiteNoise.

## Operating behind a proxy router

As shown before, OpenShift provides a platform for easily scaling up the number of instances of your web application. When this is done, OpenShift automatically handles the load balancing of requests across the multiple instance.

In order to implement this, your web application will always be run behind a proxy router. By default for OpenShift this is ``haproxy``, although it is possible to substitute other routers such as a hardware based F5.

Being behind a proxy router means that you do not see the exact HTTP request headers that the proxy router saw. Instead the request headers will reflect what is the internal address of your web application.

This is important because it means that the details as to what your web application was accessed as, when passed through to the WSGI request ``environ`` will be wrong. To accomodate this, it is necessary to consult special request headers added by the proxy router, which give details about the original host, port and scheme that the HTTP request was received on.

To deal with this you would normally need to wrap your WSGI application with a special WSGI middleware that fixed up the WSGI ``environ`` for you. That is, it is not something that would be automatically done. In a PaaS environment it is too easy to forget to do this, plus any WSGI middlemare may not do it correctly.

When using ``mod_wsgi-express`` you can avoid the need to do this yourself as ``mod_wsgi-express`` can do it for you.

To enable this feature we will provide additional options to ``mod_wsgi-express``, telling it what are the trusted proxy headers set by the proxy router. These are added to the ``.warpdrive/server_args`` file.

```
--trust-proxy-header X-Forwarded-For --trust-proxy-header X-Forwarded-Port --trust-proxy-header X-Forwarded-Scheme
```

With this in place the WSGI request ``environ`` will now be corrected even before it is passed through to your WSGI application. Importantly, any equivalent request headers often used to indicate this information will be scrubbed from the WSGI request ``environ``. This is to ensure that a HTTP client cannot try and spoof the proxy information where a WSGI middleware was present that looked for more than one possible request header.

Adding these proxy header options would also be required if relying on ``auto`` or using ``django`` with a Django application. Because ``mod_wsgi-express`` is still be used, they can be placed in the ``.warpdrive/server_args`` file in the same way.

If using ``gunicorn`` or ``waitress``, you likely will need to use a WSGI middleware to perform the fixups. Both of these WSGI servers do appear to have some options related to fixups for proxy headers, but it is unclear whether they do everything required and whether they deal properly with attempts to spoof related proxy headers by a HTTP client.