# Hacking Python for OpenShift

DEVCONF.cz 2016 - Saturday, February 6 / 10:40 - 11:20

This presentation will be in the form of a mini workshop where you will be led through performing various tasks with OpenShift related to Python web applications. If you have not participated in the prior '[Getting Started with OpenShift](https://devconfcz2016.sched.org/event/5ns0/getting-started-with-openshift)' workshop but wish to follow along, you will need to install the OpenShift CLI tools.

The setup steps for installing the OpenShift CLI tools and logging into the OpenShift environment can be found in our OpenShift Roadshow workshop notes. These can be found at:

* [http://training.runcloudrun.com/roadshow/](http://training.runcloudrun.com/roadshow/)

You should complete at least Lab 1 and then follow instructions in the Lab 2 related to logging into the OpenShift command line interface and web interface.

Direct links for these two labs are:

* [Lab 1: Installing the OpenShift CLI](http://training.runcloudrun.com/roadshow/01-install.md.html) - http://training.runcloudrun.com/roadshow/01-install.md.html
* [Lab 2: Smoke Test and Quick Tour](http://training.runcloudrun.com/roadshow/02-smoketest.md.html) - http://training.runcloudrun.com/roadshow/02-smoketest.md.html

You may wish to review at a later time other labs from the 'Getting Started with OpenShift' workshop as this presentation will not be going into depth on topics already covered in that workshop.

When ready, create a new project under your OpenShift account into which to create any applicatons described in these notes. Create the project as:

```
oc new-project userXX-python
```

Replace ``XX`` with the user number corresponding to the account you are using.

## Topics for this presentation

In this presentation you will be led through deployment of a simple Python web application to give you a basic understanding of how Python and OpenShift work together.

The presentation will then cover various features of OpenShift and implementing a custom S2I builder for Python.

The notes and instructions for each part of the presentation are:

* [Deploying a Python web application](deploying-a-python-web-application.md)
* [Application development workflow](application-development-workflow.md)
* [Selecting the Python web server](selecting-the-python-web-server.md)
* [Configuration and application scaling](configuration-and-application-scaling.md)
* [Action hooks and executing commands](action-hooks-and-executing-commands.md)

There is going to be more material here than can likely be covered in the presentation. That is in part intentional and is provided as a bit of homework you can work through later or refer to in the future.

## Trying it out all at home

If you don't wish to follow along and try the examples during the presentation, want to try it out later, but don't already have access to an OpenShift instance, you can use the All-In-One Virtual Machine for OpenShift.

* https://www.openshift.org/vm/

Alternatively, you can use Amazon Test Drive for OpenShift.

* https://aws.amazon.com/testdrive/redhat/








