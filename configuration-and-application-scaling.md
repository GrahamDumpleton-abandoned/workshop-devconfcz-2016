# Configuration and application scaling

These notes talk about manually configuring ``mod_wsgi-express`` for deployment of a Python web application.

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