# Selecting the Python web server

These notes talk about what the available Python web server options are when using the default OpenShift S2I builder for Python, as well as alternative S2I builders for Python.

## Default Python web server options

S2I builders for OpenShift can be magic, but by being magic it means that they can also be quite opinionated. Where a particular programming language has a large range of options for hosting web applications, this can mean that limited options might be allowed for what web server you can easily run.

For the case of the Python S2I builder, it steers you towards using the ``gunicorn`` WSGI server, that being the only one of the more popular Python WSGI servers that it has specific inbuilt support for.

What dictates what Python web servers can be run and when, is the ``run`` script of the Python S2I builder. You can see that for the default Python S2I builder at:

* https://github.com/openshift/sti-python/blob/master/2.7/s2i/bin/run

The checks it makes in order are as follows:

1. If an ``app.py`` file exists for the application, it is assumed to provide the complete Python web application. It will be run as ``python app.py``.
2. If the ``gunicorn`` module has been installed by being listed in the ``requirements.txt`` file, a search is made for a ``wsgi.py`` file and if  found ``gunicorn`` will be run, using the ``wsgi.py`` file as a Python module containing the WSGI application entry point. Alternatively, if there is a ``setup.py`` file, it will be used to determine the name of a module to serve as the WSGI application entrypoint. The default ``gunicorn`` configuration of a single process and single threaded ``sync`` worker will be used.
3. If Django is installed and a ``manage.py`` file exists, the Django development server will be run for a Django based application.

Unfortunately all three of these are not necessarily good defaults for various reasons.

Important to highlight as well is that there is no default WSGI server included as part of the Python S2I builder. This means that if you were to give it a WSGI 'Hello World' application where the ``requirements.txt`` file didn't exist, it would fail to start up. Thus the most basic test case one might use to test OpenShift will fail.

```
$ oc new-app python:2.7~https://github.com GrahamDumpleton/wsgi-hello-world.git
--> Found image 93d5039 (7 weeks old) in image stream "python in project openshift" under tag :2.7 for "python:2.7"
    * A source build using source code from https://github.com/GrahamDumpleton/wsgi-hello-world.git will be created
      * The resulting image will be pushed to image stream "wsgi-hello-world:latest"
    * This image will be deployed in deployment config "wsgi-hello-world"
    * Port 8080/tcp will be load balanced by service "wsgi-hello-world"
--> Creating resources with label app=wsgi-hello-world ...
    ImageStream "wsgi-hello-world" created
    BuildConfig "wsgi-hello-world" created
    DeploymentConfig "wsgi-hello-world" created
    Service "wsgi-hello-world" created
--> Success
    Build scheduled for "wsgi-hello-world" - use the logs command to track its progress.
    Run 'oc status' to view your app.
    
$ oc get pods
NAME                       READY     STATUS             RESTARTS   AGE
wsgi-hello-world-1-tppky   0/1       CrashLoopBackOff   2          30s
```

That a Python ``app.py`` script file is checked for does allow you to run any Python web server, ASYNC or WSGI, so long as it can be imported as a module and run. The problem is that the Python S2I builder does not provide PID '1' zombie reaping protection built in. If a Python web server doesn't handle reaping of zombie processes, you can run into resource usage problems, or problems with process management.

For further information on the PID '1' zombie reaping problem you can check out:

* [Issues with running as PID 1 in a Docker container](http://blog.dscpl.com.au/2015/12/issues-with-running-as-pid-1-in-docker.html) - http://blog.dscpl.com.au/2015/12/issues-with-running-as-pid-1-in-docker.html

In the case of ``gunicorn``, because no true mutithreaded worker exists, to increase capacity you have to use more processes, or scale up the number of pods. This will result in more memory resources being used. For primarily I/O bound applications, as most database backed web applications generally are, being able to use some level of multithreading allows better use of memory resources. The ``eventlet`` and ``gevent`` workers of ``gunicorn`` are not suitable as a general purpose replacement for multithreading. The ``gunicorn`` WSGI server also lacks a way of directly hosting static files.

The fallback to using the Django development server if Django is being used is the most problematic. The Django development server is not suitable for production deployments. It can only handle one request at a time, does not handle the PID '1' zombie reaping problem, as well as having other potential performance issues with the way it handles static files and due to its code reloading mechanism.

One could technically run other command line Python web servers by implementing an ``app.py`` which uses ``os.execl()`` to execute them, but when using a S2I builder you are prevented from installing additional system packages. This is an issue because the default Python S2I builder doesn't install the Apache HTTPD server, which would be required if for example you wanted to run the ``mod_wsgi-express`` server.

## Alternative Python S2I builders

The OpenShift S2I Python builder will hopefully be improved over time, offering a more general purpose solution to support alternate production grade Python web servers, as well as built in protection for the PID '1' zombie reaping problem.

Even so, as the expertise as to what is the best way to setup a Docker container for hosting Python web applications, as well as knowledge of the best server options and configuration likely lies within the wider Python web community, it would be an ideal opportunity for the Python web community to come up with a best of bread S2I builder for Python themselves and drive its development.

One such proof of concept implementation for an alternative S2I builder for Python which is available is based around the ``warpdrive`` project:

* https://github.com/GrahamDumpleton/warpdrive-python

This project provides scripts to assist in building up a Docker image containing a Python web application and then starting it up when the container is run. S2I ``assemble`` and ``run`` scripts are also provided which then allow a Docker base image for Python web applications to be S2I enabled using the ``warpdrive`` scripts.

The two layers of the ``warpdrive`` scripts and the S2I scripts, means that the Docker base image for a Python web application can be used directly independent of S2I, or using S2I directly or in conjunction with OpenShift.

Currently pre-built Docker base images using ``warpdrive`` are made available on Docker Hub as:

* [grahamdumpleton/warp0-debian8-python27](https://hub.docker.com/r/grahamdumpleton/warp0-debian8-python27/)
* [grahamdumpleton/warp0-debian8-python35](https://hub.docker.com/r/grahamdumpleton/warp0-debian8-python35/)

Image streams suitable for OpenShift can be created for these using the command:

```
oc create -f https://raw.githubusercontent.com/GrahamDumpleton/warp0-debian8-python/master/openshift.json
```

The local image stream name and tags created for these will be:

* warp0-debian8-python:2.7
* warp0-debian8-python:3.5

To use these S2I builders, search for ``warpdrive`` in the OpenShift UI when adding an application to a project. Then supply the URL for the Git repository for the Python web application.

Alternatively, you can use the ``oc new-app`` command.

Unlike the default Python S2I builder, these S2I builders come with the Apache HTTPD server and ``mod_wsgi-express`` already installed. This means that the most basic test case of a WSGI 'Hello World' application will work out of the box.

```
$ oc new-app warp0-debian8-python:2.7~https://github.com/GrahamDumpleton/wsgi-hello-world.git
--> Found image f67ce21 (30 minutes old) in image stream "warp0-debian8-python" under tag :2.7 for "warp0-debian8-python:2.7"
    * A source build using source code from https://github.com/GrahamDumpleton/wsgi-hello-world.git will be created
      * The resulting image will be pushed to image stream "wsgi-hello-world:latest"
    * This image will be deployed in deployment config "wsgi-hello-world"
    * Port 8080/tcp will be load balanced by service "wsgi-hello-world"
--> Creating resources with label app=wsgi-hello-world ...
    ImageStream "wsgi-hello-world" created
    BuildConfig "wsgi-hello-world" created
    DeploymentConfig "wsgi-hello-world" created
    Service "wsgi-hello-world" created
--> Success
    Build scheduled for "wsgi-hello-world" - use the logs command to track its progress.
    Run 'oc status' to view your app.
    
$ oc get pods
NAME                       READY     STATUS      RESTARTS   AGE
wsgi-hello-world-1-t4wh0   1/1       Running     0          19s

$ oc expose service wsgi-hello-world
route "wsgi-hello-world" exposed
```

Our Django example will also work fine as well, again using ``mod_wsgi-express`` where as with the default Python S2I builder it was using the Django development server.

```
$ oc new-app warp0-debian8-python:2.7~https://github.com/GrahamDumpleton/django-hello-world-v1
--> Found image ac4701a (About an hour old) in image stream "warp0-debian8-python" under tag :2.7 for "warp0-debian8-python:2.7"
    * A source build using source code from https://github.com/GrahamDumpleton/django-hello-world-v1 will be created
      * The resulting image will be pushed to image stream "django-hello-world-v1:latest"
    * This image will be deployed in deployment config "django-hello-world-v1"
    * Port 8080/tcp will be load balanced by service "django-hello-world-v1"
--> Creating resources with label app=django-hello-world-v1 ...
    ImageStream "django-hello-world-v1" created
    BuildConfig "django-hello-world-v1" created
    DeploymentConfig "django-hello-world-v1" created
    Service "django-hello-world-v1" created
--> Success
    Build scheduled for "django-hello-world-v1" - use the logs command to track its progress.
    Run 'oc status' to view your app.
    
$ oc expose service django-hello-world-v1
route "django-hello-world-v1" exposed
```

## Automatic versus explicit setup

If you are going to supply some sort of magic automatic option in a Python S2I builder for deploying a WSGI application without you needing to setup the WSGI server, then it has to work well. When making guesses you need to be pretty sure that you are correct. If you are going to dictate how the web application is started, then the web server you use needs to be something that is suitable for production use.

Even if you can make a pretty good system for automatically working out what needs to be done, there must be a simple way of disabling any automatic configuration and allowing a user to take full control.

Even though a user taking over configuration is the preferred option so they understand what is being done, the ``warpdrive`` based Python S2I builder still provides an automatic option purely because people like to see things work magically out of the box. A PaaS provider also likes such an automatic mode as it gives the appearance that things are easy to deploy using their service.

For the ``warpdrive`` based Python S2I builder the way that automatic deployment works is as follows:

1. If an ``app.sh`` file exists for the application, it will assume that this is a shell script which will take full control of starting up an appropriate web server for the Python web application. Configuration mode will be switched from ``auto`` to ``shell`` mode.
2. If an ``app.py`` file exists for the application, it will assume that this is a Python script which will take full control of starting up an appropriate web server for the Python web application. Configuration mode will be switched from ``auto`` to ``python`` mode.
3. If a ``paste.ini`` file exists for the application, it will assume that a Paste style configuration file defines the WSGI application. Configuration mode will be switched from ``auto`` to ``mod_wsgi`` and ``mod_wsgi-express`` given options to host the WSGI application described by the Paste configuration file.
4. If a ``wsgi.py`` file exists for the application, it will assume this is a WSGI script file containing the WSGI application entry point. Configuration mode will be switched from ``auto`` to ``mod_wsgi`` and ``mod_wsgi-express`` given options to host the WSGI application in the WSGI script file. For compatibility with OpenShift 2, if the directory ``wsgi/static`` exists, it will also be served up as static files under the URL ``/static``.
5. For compatibilty with OpenShift 2, if a ``wsgi/application`` file exists for the application, it will assume this is a WSGI script file containing the WSGI application entry point. Configuration mode will be switched from ``auto`` to ``mod_wsgi``  and ``mod_wsgi-express`` given options to host the WSGI application in the WSGI script file. Again for compatibility with OpenShift 2, if the directory ``wsgi/static`` exists, it will also be served up as static files under the URL ``/static``.
6. If a ``setup.py`` file exists for the application, it will assume that was used to install an application module which should be used as the WSGI application. The name of the module is queried from ``setup.py``. Configuration mode will be switched from ``auto`` to ``mod_wsgi`` and ``mod_wsgi-express`` given options to host the WSGI application provide by the application module.
7. If a ``manage.py`` file exists for the application, it will validate whether this is the Django management script. If it is, the details of the WSGI application entry point and location of static files will be queried using ``manage.py``. Configuration mode will be switched from ``auto`` to ``mod_wsgi`` and ``mod_wsgi-express`` given options to host the Django application including static files.

The automatic deployment mechanism therefore provides many more options than the default Python S2I builder. This includes being able to make use of the bundled WSGI server provided by ``mod_wsgi-express``. Compatibility is also provided for various legacy ways which a WSGI application could be setup under OpenShift 2, helping to make a transiton to OpenShift 3 easier.

If the automatic mechanism isn't sufficient, or a much greater level of configuration control is required, the automatic mechanism can be switched off and the specific type of server mechanism to be used specified.

In addition to all this, the ``warpdrive`` project provides PID '1' zombie reaping support itself in case a Python web application itself doesn't look after that. All the usual tricks required to have a Docker image run as an arbitrary user ID under OpenShift are also all incorporated into the Docker image.


