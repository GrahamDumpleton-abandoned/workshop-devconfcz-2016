# Deploying a Python web application

These notes cover deployment of a Python web application using the [Source to Image](https://github.com/openshift/source-to-image) feature of OpenShift.

## Application deployment options

For any application being installed to OpenShift the main options you have are:

* Use a Docker image for an application which has been built separate to OpenShift and which is available on a remote Docker registry.
* Have OpenShift build a Docker image from a Git repository containing a ``Dockerfile`` and the application code. The resulting Docker image would then be used.
* Have OpenShift use a Source to Image (S2I) builder to combine application code from a Git repository with the S2I builder base Docker image containing a run time environment for the language of the application. The resulting Docker image would then be used.

The S2I approach is the quickest way to get started as you as a developer do not have to worry about all the steps needed to set up a suitable runtime environment for your language. You can instead rely on the S2I builder for your language provided by OpenShift, or even your own operations team or another third party.

The use of a S2I builder image ensures that what you get incorporates all the best practices around setting up images for OpenShift and also for running applications using your language of choice in an OpenShift environment.

We will be using the S2I method for handling deploying of a Python web application.

## Sample Python web application

For our initial example we are going to use a simple Django 'Hello World' application. The Git repository for this can be found at:

* [https://github.com/GrahamDumpleton/django-hello-world-v1](https://github.com/GrahamDumpleton/django-hello-world-v1)

To deploy this Django web application to OpenShift we use the ``oc new-app`` command:

```
$ oc new-app https://github.com/GrahamDumpleton/django-hello-world-v1.git
--> Found image bae1743 (6 weeks old) in image stream "python in project openshift" under tag :latest for "python"
    * The source repository appears to match: python
    * A source build using source code from https://github.com/GrahamDumpleton/django-hello-world-v1.git will be created
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
```

The only required argument we provide to ``oc new-app`` in this case is the Git repository URL.

Initially the resulting web application is not publicly accessible, so once deployed we also need to expose it to the Internet.

```
$ oc expose service django-hello-world-v1
route "django-hello-world-v1" exposed
```

The default public hostname and URL assigned to the web application by OpenShift can be found via the web interface to OpenShift, or using the ``oc describe route`` command:

```
$ oc describe route django-hello-world-v1
Name:			django-hello-world-v1
Created:		15 minutes ago
Labels:			app=django-hello-world-v1
Annotations:		openshift.io/host.generated=true
Host:			django-hello-world-v1-devconfcz-workshop.apps.example.com
Path:			<none>
Service:		django-hello-world-v1
TLS Termination:	<none>
Insecure Policy:	<none>
```

## Python language detection

When using ``oc new-app`` against the Git repository we did not specify that it was in fact a Phython web application. The ``oc new-app`` command will work out what programming language the web application is implemented in by looking at what files are contained in the Git repository.

For Python web applications, ``oc new-app`` will look for the presence of the following files:

* ``requirements.txt`` - List of Python packages which would be installed using the ``pip`` command.
* ``setup.py`` - A Python setup file for installing an application as a Python package. (OSE 1.1.1 or later only)

If either file exists it will assume that the Git repository contains a Python wbe application and select the Python S2I builder to create a Docker image for the web application and deploy it.

OpenShift current supports S2I builders for the following Python versions:

* Python 2.7
* Python 3.3
* Python 3.4

When automatic language detection is used and Python detected, it will use Python 3.4.

If you wished to use Python 2.7, you would need to have instead indicated that Python was to be used as well as what specific version of Python. This would have been done using the command:

```
oc new-app python:2.7~https://github.com/GrahamDumpleton/django-hello-world-v1.git
```


