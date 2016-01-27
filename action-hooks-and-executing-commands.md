# Action hooks and executing commands

These notes talk about how to perform additional actions when the Docker image for your application is being built, deployed and when it is running.

## Application actions hooks

The ``assemble`` and ``run`` scripts of the Python S2I builder, used during the image build and deployment phases, are self contained and offer no way for a user to specify additional actions to be performed. This is extremely limiting as it means that if the S2I builder doesn't do something automatically for you, then you are stuck and wouldn't be able to use it.

As an example, when building an image for a Django application it is necessary to trigger the collection of static file assets from their various locations and place them into a single directory so they can be served by a web server. This is done using the Django management command called ``collectstatic``.

The default Python S2I builder will trigger ``collectstatic`` if it detects that you are likely using Django as does the ``warpdrive`` based Python S2I builder.

In the case of the ``warpdrive`` builder, this will only be done if using ``auto`` or ``django`` server types. If you specify ``mod_wsgi`` with the intent of taking over full control, then it doesn't automatically run ``collectstatic`` for you.

In order to allow you to specify that ``collectstatic`` or any other extra steps should be run during image build or deployment, the ``warpdrive`` based Python S2I builder provides a hook mechanism whereby you can do whatever you need at key phases.

This hooking mechanism is modelled after what was done in OpenShift 2, a feature that has been dropped from the default OpenShift 3 Python S2I builder.

The available action hooks are:

* ``pre-build`` - Triggered during an image build, prior to any Python packages from the ``requirements.txt`` file being installed using ``pip``. This hook could be used to install any custom third party packages that may be needed when installing a Python package.
* ``build`` - Triggered during an image build, after any Python packages from the ``requirements.txt`` file have been installed using ``pip``. This hook would be used to prepare your application code ready to be run.
* ``deploy-env`` - Triggered when the container is first started. This hook would be used to setup any additional environment variables which may be dependent on per instance environment information.
* ``deploy`` - Triggered in the running container just prior to the actual web application being started. This hook would be used to setup any per instance data required by the application.

## Collecting static files for Django

For the case of Django, if you are taking over full control of configuration by specifying the server type as ``mod_wsgi`` or another WSGI server, you need to trigger collection of the static files yourself.

To have this be triggered as part of the ``build`` phase when the S2I builder is run we add an executable script file ``.warpdrive/action_hooks/build`` and place in it:

```
#!/bin/bash

set -x

python manage.py collectstatic --noinput
```

## Installation of system packages

When using an S2I builder, the image build runs as a non privileged user. This means you cannot install system packages.

To get around this you would need to create a custom S2I builder. Generally when this is necessary people clone the existing S2I builder they want to base things on and modify it directly. With the ``warpdrive`` based Python S2I builder, it is relatively easy to create a derived image.

For example, to create a S2I builder for running IPython notebooks the ``Dockerfile`` for the derived image would be:

```
FROM grahamdumpleton/warp0-python27-debian8

USER 0

RUN apt-get update && apt-get install -y libfreetype6 libfreetype6-dev \
    libpng++ libpng++-dev liblapack-dev libatlas-dev gfortran && \
    rm -r /var/lib/apt/lists/*

USER 1001

RUN pip install --no-cache-dir jupyter ipython[notebook] matplotlib \
    sympy ipyparallel scipy

COPY ipython/ /home/warpdrive/ipython/
COPY jupyter/ /home/warpdrive/jupyter/

COPY app.sh /home/warpdrive/app.sh

ENV WARPDRIVE_SHELL_FILE=/home/warpdrive/app.sh

EXPOSE 8080

EXPOSE 10000-10011
```

In order to install the actual system packages, you need to switch to ``USER 0`` so they are installed as ``root``.

We also want to install IPython itself as part of the base image as well as installing IPython takes too long to be performed on each build. This is installed after switching back to the S2I builder user so that reasonable file permissions are maintained.

As IPython requires a custom startup script, we can use the ability of the ``warpdrive`` based Python image to allow us to override startup using our own shell script.

As it happens in this case, the resulting Docker image can be used by itself, run directly to create a instance of IPython.

```
$ oc new-app grahamdumpleton/s2i-ipython-notebook:python-2.7 --name nbviewer1
--> Found Docker image 51a5860 (14 hours old) from Docker Hub for "grahamdumpleton/s2i-ipython-notebook:python-2.7"
    * An image stream will be created as "s2i-ipython-notebook:python-2.7" that will track this image
    * This image will be deployed in deployment config "nbviewer1"
    * Ports 10000/tcp, 10001/tcp, 10002/tcp, 10003/tcp, 10004/tcp, 10005/tcp, 10006/tcp, 10007/tcp, 10008/tcp, 10009/tcp, 10010/tcp, 10011/tcp, 8080/tcp will be load balanced by service "nbviewer1"
--> Creating resources with label app=nbviewer1 ...
    ImageStream "nbviewer1" created
    DeploymentConfig "nbviewer1" created
    Service "nbviewer1" created
--> Success
    Run 'oc status' to view your app.

$ oc expose service nbviewer1
route "nbviewer1" exposed
```

Alternatively, it is used as an S2I builder to create an instance which incorporates IPython notebooks from a Git repository. That same Git repository can even have a ``requirements.txt`` file with additional Python packages to be installed which are required by the IPython notebooks.

```
$ oc new-app grahamdumpleton/s2i-ipython-notebook:python-2.7~https://github.com/jrjohansson/scientific-python-lectures.git --name nbviewer2
--> Found Docker image 51a5860 (14 hours old) from Docker Hub for "grahamdumpleton/s2i-ipython-notebook:python-2.7"
    * An image stream will be created as "s2i-ipython-notebook:python-2.7" that will track this image
    * A source build using source code from https://github.com/jrjohansson/scientific-python-lectures.git will be created
      * The resulting image will be pushed to image stream "nbviewer2:latest"
      * Every time "s2i-ipython-notebook:python-2.7" changes a new build will be triggered
    * This image will be deployed in deployment config "nbviewer2"
    * Ports 10000/tcp, 10001/tcp, 10002/tcp, 10003/tcp, 10004/tcp, 10005/tcp, 10006/tcp, 10007/tcp, 10008/tcp, 10009/tcp, 10010/tcp, 10011/tcp, 8080/tcp will be load balanced by service "nbviewer2"
--> Creating resources with label app=nbviewer2 ...
    error: imageStream "s2i-ipython-notebook" already exists
    ImageStream "nbviewer2" created
    BuildConfig "nbviewer2" created
    DeploymentConfig "nbviewer2" created
    Service "nbviewer2" created

$ oc expose service nbviewer2
route "nbviewer2" exposed
```

## Executing commands inside of a container

To get access to an interactive shell inside of the container you can use ``oc rsh``.

```
$ oc get pods --selector app=django-hello-world-v1
NAME                            READY     STATUS    RESTARTS   AGE
django-hello-world-v1-2-ek830   1/1       Running   0          19h

$ oc rsh django-hello-world-v1-2-ek830
I have no name!@django-hello-world-v1-2-ek830:/app$
```

Note how the prompt says '``I have no name!``'.

This is due to the fact that OpenShift runs your container as a user ID which is neither root, nor the S2I builder user ID of ``1001`` used above. The unique user ID OpenShift assigns to your project does not reside in the system password database for the operating system within your Docker image.

This can cause problems for any application you run which doesn't handle gracefully that it is running as a user ID with no associated UNIX account.

When your actual web application is deployed, such problems are avoided as the ``warpdrive`` scripts will use a package called ``nss_wrapper`` which intercepts requests by applications to look up UNIX user account information and return fake information for the assigned user ID.

When you use ``oc rsh`` and ``bash`` is run, the setup for ``nss_wrapper`` isn't however done, so you are faced with this problem.

In order to get the same environment as what your deployed application will be using, so that manually run commands will work, or even to validate what environment the application is running with, you can use the command ``warpdrive-shell`` with ``oc rsh``:

```
$ oc rsh django-hello-world-v1-2-ek830 warpdrive-shell
warpdrive@django-hello-world-v1-2-ek830:/app$
```

Note that as well as setting up ``nss_wrapper``, this will also trigger any ``deploy-env`` action hook so that the shell has the same environment as the deployed application.

If you didn't want an interactive shell, but just wanted to run a command inside of the container and know it used the same environment, then you can use ``oc exec`` and the ``warpdrive-exec`` command.

```
$ oc exec django-hello-world-v1-2-ek830 warpdrive-exec python manage.py migrate
Operations to perform:
  Apply all migrations: sessions, auth, contenttypes, admin
Running migrations:
  Rendering model states... DONE
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying sessions.0001_initial... OK
```

The ``warpdrive-exec`` command would also be used if needing to run the same application image as a once off job to perform database migration during deployment.

Use of a job like this as a separate phase is better than trying to implement database migration as a ``deploy`` action hook. In fact, a ``deploy`` hook should never be used for database migration as it will only cause problems, especially when a pod is scaled up.

Because of that, it is very important if using the default Python S2I builder that you always disable the database migration it triggers for Django automatically. This is done by setting the ``DISABLE_MIGRATE`` environment variable when creating your application.

The ``warpdrive`` based Python S2I builder does not automatically trigger database migrations due to the danger of running them except in a controlled way dependent on how your application is deployed.