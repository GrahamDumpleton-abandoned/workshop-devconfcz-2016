# Application development workflow

These notes talk about the developer workflow for working on a Python web application being deployed to OpenShift.

## Local development environment

Traditional development workflows for Python web applications would still be followed. You can therefore still develop your code changes on your local system.

In this case we are using the Django web framework and it comes with its own builtin development server. To develop locally on your own system you can therefore setup an appropriate Python virtual environment with all the packages listed in your ``requirements.txt`` file and use:

```
$ python manage.py runserver
Performing system checks...

System check identified no issues (0 silenced).
January 21, 2016 - 02:49:10
Django version 1.9.1, using settings 'hello_world.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

Being the Django development server, application reloading will automatically occur as code changes are being made.

## Publishing your code changes

When your Python web application was deployed to OpenShift, it was given the Git repository URL. For any change to make it into your web application, they would by default need to be commited into your local Git repository and then pushed back up to the remote Git repository. A rebuild and redeployment of your web application on OpenShift would then need to be triggered.

The name of our web application within OpenShift was ``django-hello-world-v1``. Once a change has been pushed back to the remote repository, the rebuild and redeployment can therefore be triggered using:

```
$ oc start-build django-hello-world-v1
django-hello-world-v1-2
```

Because the application in OpenShift was created from what will be to you a read only Git repository, if you want to test the effect of code changes yourself you can fork the Git repository. You will though need to either destroy the original application and recreate it from the new Git repository, or update the build configuration for your application to use the new Git repository.

To delete the original application from OpenShift you can use the command:

```
$ oc delete all --selector app=django-hello-world-v1
buildconfig "django-hello-world-v1" deleted
build "django-hello-world-v1-1" deleted
build "django-hello-world-v1-2" deleted
imagestream "django-hello-world-v1" deleted
deploymentconfig "django-hello-world-v1" deleted
route "django-hello-world-v1" deleted
service "django-hello-world-v1" deleted
pod "django-hello-world-v1-2-sct5r" deleted
```

Then recreate the application using ``oc new-app`` and your fork of the Git repository.

To re-point the already deployed application to your fork of the Git repository. You can do this using the ``oc edit`` command:

```
oc edit buildconfig django-hello-world-v1
```

This will throw you into an editor. Change the ``uri`` for the Git repository to the new location, save and exit the editor.

```
  source:
    git:
      uri: https://github.com/GrahamDumpleton/django-hello-world-v1.git
    type: Git
```

When a change is made to the build configuration in this way, a new build and deployment will be automatically triggered. After subsequent code changes you can then use ``oc start-build``.

## Automated code deployments

If you prefer a more automated end to end solution where your Python web application is automatically rebuilt and reployed when code changes are pushed up to your remote Git repository, you can enable WebHooks integration with OpenShift.

Further information about this can be found in Lab 8 of the 'Getting Started with OpenShift' workshop notes.

* [Lab 8: Making Code Changes and using webhooks](http://training.runcloudrun.com/roadshow/08-codechanges.md.html) - http://training.runcloudrun.com/roadshow/08-codechanges.md.html

You can find the WebHook URL to configure into GitHub from the build configuration view in the OpenShift web interface, or by using the ``oc describe buildconfig`` command:

```
$ oc describe buildconfig django-hello-world-v1
Name:			django-hello-world-v1
Created:		35 minutes ago
Labels:			app=django-hello-world-v1
Annotations:		openshift.io/generated-by=OpenShiftNewApp
Latest Version:		2
Strategy:		Source
Source Type:		Git
URL:			https://github.com/GrahamDumpleton/django-hello-world-v1.git
From Image:		ImageStreamTag openshift/python:latest
Output to:		ImageStreamTag django-hello-world-v1:latest
Triggered by:		Config, ImageChange
Webhook GitHub:		https://openshift-master.example.com:443/oapi/v1/namespaces/devconfcz-workshop/buildconfigs/django-hello-world-v1/webhooks/FKDBMz-dhyxYaapCMUvL/github
Webhook Generic:	https://openshift-master.example.com:443/oapi/v1/namespaces/devconfcz-workshop/buildconfigs/django-hello-world-v1/webhooks/klVHzOpfsyHLMZGYKaJi/generic

Build				Status		Duration	Creation Time
django-hello-world-v1-2 	complete 	43s 		2016-01-21 14:06:54 +1100 AEDT
django-hello-world-v1-1 	complete 	56s 		2016-01-21 14:02:02 +1100 AEDT
```

Once configured, any push of changes to your remote Git repository will automatically result in your web application being rebuilt and redeployed.

## Bypassing the remote Git repository

If your local development environment isn't a valid equivalent of the OpenShift environment where your Python web application is being deployed, it may not always be possible to exactly replicate and test an issue in your local development environment.

This means that you may have to commit and push up changes to your remote Git repository and rebuild and redeploy your web application to properly see the results of a change.

This requirement to commit and push changes in this sort of scenario can quickly become cumbersome and can result in a messy commit history.

If the instance of your application in OpenShift is specifically for development and testing, there are ways of bypassing the remote Git repository and so get a faster turn around on changes without leaving a trail of commits in your Git repository.

The first such way to achieve this is to indicate to OpenShift when triggering a build, to use the source files for your application from a local directory on your own system, rather than from the Git repository.

To do this, from within the top level directory of the checkout of your Git repository, run:

```
 oc start-build django-hello-world-v1 --from-dir .
Uploading "." at commit "HEAD" as binary input for the build ...
Uploading directory "." as binary input for the build ...
django-hello-world-v1-3
```

What ``oc start-build`` does when supplied ``--from-dir`` is package up the local source directory and send it up to OpenShift. That will be used instead of the contents of the Git repository. It will use whatever files are in the directory, including files containing changes that have not been commited into the Git repository.

Note that the use of the local directory will only apply to that one build. To rebuild with it reverting back to what is in the remote Git repository, trigger a new build without the ``--from-dir`` option:

```
$ oc start-build django-hello-world-v1
django-hello-world-v1-4
```

## Making live source code changes

A second way for quickly making changes is to get inside the Docker container where your Python web application is running and edit the live source code.

To detemine the name assigned to the Docker container, you can use the ``oc get pods`` command:

```
$ oc get pods
NAME                            READY     STATUS      RESTARTS   AGE
django-hello-world-v1-4-0g090   1/1       Running     0          13m
```

An interactive shell within the running Docker container can then be started using ``oc rsh``:

```
$ oc rsh django-hello-world-v1-4-0g090
bash-4.2$ ls -las
total 60
 4 drwxrwxrwx. 4 default    root  4096 Jan 20 23:12 .
 0 drwxrwxrwx. 4 default    root    26 Dec  3 03:41 ..
 4 -rw-------. 1 1000230000 root    41 Jan 20 23:12 .bash_history
 4 -rw-rw-r--. 1 default    root   747 Jan 20 22:58 .gitignore
 0 drwxrwx---. 4 default    root    26 Jan 20 22:58 .local
 4 -rw-rw-r--. 1 default    root  1299 Jan 20 22:58 LICENSE
36 -rw-r--r--. 1 1000230000 root 36864 Jan 20 22:59 db.sqlite3
 0 drwxrwxr-x. 3 default    root   103 Jan 20 23:10 hello_world
 4 -rwxrwxr-x. 1 default    root   254 Jan 20 22:58 manage.py
 4 -rw-rw-r--. 1 default    root    14 Jan 20 22:58 requirements.txt
```

Make your code changes and the Python web application will perform an internal restart and pick up the new changes.

Note that this only works because for our Django application, the default OpenShift Python S2I builder has decided to run the builtin Django development server.

Although it seems to be a favourite of PaaS offerings to automatically use the Django development server to make it appear that installation is really simple, it is never recommended that the Django development server be used in production. It is arguable that even having support for the Django development server as a default is a very bad idea and the option should be removed.

Also be aware that live source code changes will only work where some other action doesn't need to be taken, such as installing additional Python packages. Depending on the Docker base image being used and the application itself, you may or may not be able to install additional packages manually in the running Docker container.

Finally, making live source code changes only make sense where you have a single instance of your web application running. This is because the application code is local to each Docker container. Where multiple instances are running, subsequent requests may not be handled by the Docker container where you made the live code changes.

## Sycnhronising files with a pod

A variation on making live coding changes is that rather than enter into the running Docker container, you make the changes in your local directory on your own system. You can then synchronise the files with a pod. This is done using ``oc rsync``:

```
$ oc rsync ./hello_world django-hello-world-v1-4-0g090:/opt/app-root/src/hello_world
building file list ... done
hello_world/
hello_world/__init__.py
hello_world/__init__.pyc
hello_world/settings.py
hello_world/settings.pyc
hello_world/urls.py
hello_world/urls.pyc
hello_world/views.py
hello_world/views.pyc
hello_world/wsgi.py
hello_world/wsgi.pyc

sent 10156 bytes  received 246 bytes  2972.00 bytes/sec
total size is 9476  speedup is 0.91
```

Your mileage on this may vary as although it will synchronise files across, the ``rsync`` command does appear to give strange errors at times depending on what directories you are syncing and the permissions on those directories.

Either way, because the Django development server was being used, your Python web application will again be internally restarted when the changed code files are detected.

Files can also be copied back out of a running Docker container using ``oc rsync`` if you had instead use ``oc rsh`` to enter into the Docker container to make changes and wish to save those changes.

Do remember that a running Docker container is ephemeral and the application code is not stored on a persistent disk volume. If making changes in the running Docker container itself and the container were shutdown and restarted, you will loose your changes.
