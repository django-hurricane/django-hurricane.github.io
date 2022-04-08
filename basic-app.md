---
layout: page
title: Guide to your first Hurricane-based Application
---
## What you'll learn
- Create a Django app with Hurricane
- Run it locally with virtualenv
- Run it in local Kubernetes with k3d

## What you'll need
- [poetry](https://python-poetry.org/docs#installation){:target="_blank"}
- [k3d](https://k3d.io/v5.4.1/#installation){:target="_blank"}
- [helm](https://helm.sh/docs/intro/install/){:target="_blank"}
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl){:target="_blank"}

## Table of contents
1. [Application details and settings](#1-application-details-and-settings)
   1. [Spacecrafts application and Hurricane settings](#spacecrafts-application-and-hurricane-settings)
   2. [Details of GraphQL configuration](#details-of-graphql-configuration)
2. [2. Run this app in a local virtualenv](#2-run-this-app-in-a-local-virtualenv)
   1. [Set environment variables](#set-environment-variables)
   2. [Start the Hurricane server](#start-the-hurricane-server)
3. [Run this application using Django-Hurricane in a Kubernetes cluster](#3-run-this-application-using-django-hurricane-in-a-kubernetes-cluster)
   1. [Creating the k3d cluster](#creating-the-k3d-cluster)
   2. [Install the Helm charts](#install-the-helm-charts)
   3. [Local Kubernetes development: code hot-reloading, debugging and more](#local-kubernetes-development-code-hot-reloading-debugging-and-more)

## 1. Application details and settings

### Spacecrafts application and Hurricane settings
We will make it a bit more interesting and instead of a plain Django application we will create a django-graphene application.
You can learn more about Graphene [<ins>**here**</ins>](https://docs.graphene-python.org/projects/django/en/latest/){:target="_blank"}. In its essence, with 
django-graphene we will be able to use GraphQL functionality for our application.

You can find the application at [<ins>**this repository**</ins>](https://github.com/django-hurricane/spacecrafts-demo){:target="_blank"}.
It is set up with our [<ins>**cookiecutter template**</ins>](https://github.com/django-hurricane/django-hurricane-template){:target="_blank"}.

Our spacecrafts app has two simple models to configure components that are associated to a category. 
They are contained in a Django app called `components`.
The `models.py` looks like this:
~~~python
# src/apps/components/models.py

from django.db import models


class Category(models.Model):
    title = models.CharField(
        max_length=100, 
    )

    def __str__(self):
        return self.title


class Component(models.Model):
    title = models.CharField(max_length=100)
    description = models.TextField()
    category = models.ForeignKey(
        Category, 
        related_name="categories", 
        on_delete=models.CASCADE,
    )

    def __str__(self):
        return self.title

~~~

The `components`-app as well as `graphene_django` and `hurricane` are added in the settings file:

~~~python
# src/configuration/components/apps.py

INSTALLED_APPS = [
    ...,
    "apps.components.apps.ComponentsConfig",
    "graphene_django",
    "hurricane",
]
~~~

The logging settings are adapted to have Hurricanes logs available:
~~~python
# src/configuration/components/logging.py

LOGGING = {
    "version": 1,
    "disable_existing_loggers": True,
    "formatters": {"console": {"format": "%(asctime)s %(levelname)-8s %(name)-12s %(message)s"}},
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "console",
            "stream": sys.stdout,
        }
    },
    "root": {"handlers": ["console"], "level": "INFO"},
    "loggers": {
        "hurricane": {
            "handlers": ["console"],
            "level": os.getenv("HURRICANE_LOG_LEVEL", "INFO"),
            "propagate": False,
        },
    },
}

~~~

### Details of GraphQL configuration

The essential part of `graphene_django` is a graph representation of objects. Such a representation is defined in a so-called schema, which can be found in `schema.py`:

~~~python
# src/configuration/schema.py

import graphene
from graphene_django import DjangoObjectType

from apps.components.models import Category, Component


class CategoryType(DjangoObjectType):
    class Meta:
        model = Category
        fields = ("id", "title")


class ComponentType(DjangoObjectType):
    class Meta:
        model = Component
        fields = ("id", "title", "description", "category")


class Query(graphene.ObjectType):
    all_components = graphene.List(ComponentType)
    category_by_name = graphene.Field(CategoryType, name=graphene.String(required=True))

    def resolve_all_components(root, info):
        return Component.objects.select_related("category").all()

    def resolve_category_by_name(root, info, name):
        try:
            return Category.objects.get(name=name)
        except Category.DoesNotExist:
            return None


schema = graphene.Schema(query=Query)

~~~
> For more information on the concept of schema, please refer to [<ins>**Graphene Documentation**</ins>](https://docs.graphene-python.org/projects/django/en/latest/schema/){:target="_blank"}.

The settings are configured to let Django know where it can find the graphene schema:
~~~python
# src/configuration/components/graphene.py

GRAPHENE = {
    "SCHEMA": "configuration.schema.schema"
}
~~~


Additionally, the graphql endpoint is added to the urls configuration:
~~~python
# src/configuration/urls.py

from django.contrib import admin
from django.urls import path
from django.views.decorators.csrf import csrf_exempt

from graphene_django.views import GraphQLView

urlpatterns = [
    path("admin/", admin.site.urls),
    path("graphql", csrf_exempt(GraphQLView.as_view(graphiql=True))),
]
~~~


## 2. Run this app in a local virtualenv

### Set environment variables

You need to set some environment variables, you can do that by creating a `.env`-file in `src` with following content:
~~~bash
# src/.env
DATABASE_ENGINE=django.db.backends.sqlite3
DATABASE_NAME=spacecrafts.sqlite3
DJANGO_SECRET_KEY=H96HwkhWFCKmWjRnJKJNkT3wSDCJ7MJ22Qi5C5t9UX8Hem89Q4
DJANGO_STATIC_ROOT=static
DJANGO_DEBUG=True
~~~

### Start the Hurricane server
Before you can start the server you need to activate a virtualenv, and install the dependencies:
~~~bash
poetry shell
poetry install
~~~

Now you can start the server. With `--autoreload` flag server will be automatically reloaded upon changes in the code. 
Static files will be served if you add `--static` flag. We instruct Hurricane to run two Django management command. We collect statics with `--command 'collectstatic --noinput'` and we also migrate the database with `--command 'migrate'`.
~~~bash
python manage.py serve --autoreload --static --command 'collectstatic --noinput' --command 'migrate'
~~~

You should get similar output upon the start of the server:
~~~bash
2022-01-21 10:19:21,434 INFO     hurricane.server.general Tornado-powered Django web server
2022-01-21 10:19:21,435 INFO     hurricane.server.general Autoreload was performed
2022-01-21 10:19:21,435 INFO     hurricane.server.general Starting probe application running on port 8001 with route liveness-probe: /alive, readiness-probe: /ready, startup-probe: /startup
[...]
2022-01-21 10:19:21,436 INFO     hurricane.server.general Starting HTTP Server on port 8000
2022-01-21 10:19:21,436 INFO     hurricane.server.general Serving static files under /static/ from static
2022-01-21 10:19:21,437 INFO     hurricane.server.general Startup time is 0.0026073455810546875 seconds
~~~

Our repository contains a fixture to get your spacecrafts components started, you can load it by running in a `poetry shell`:
~~~bash
python manage.py loaddata components
~~~
Alternatively you can create model instances on your own through the admin interface.

After going to the graphql url ([http://127.0.0.1:8000/graphql](http://127.0.0.1:8000/graphql){:target="_blank"}) you can play around with GraphQL querying. For example you could list all components and their categories:
~~~
{
  allComponents {
    id
    title
    description
    category {
      id
      title
    }
  }
}
~~~


In addition to the previously defined `admin` and `graphql` endpoints, Hurricane starts a probe server on port+1, unless an explicit port for probes is specified. This feature is essential for cloud-native development, and it is only one of the many features of Django-Hurricane. For further features and information on Hurricane, please refer to [<ins>**Full Django Hurricane Documentation**</ins>](https://django-hurricane.readthedocs.io/en/latest/){:target="_blank"}.


## 2. Run this application in a Kubernetes cluster

### Creating the k3d cluster
We're using [k3d](https://k3d.io/){:target="_blank"} to create and run a local kubernetes cluster. You can install it via:
~~~bash
wget -q -O - https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash
~~~

After installing k3d you can create a spacecrafts cluster with the following command:

~~~bash
k3d cluster create spacecrafts --agents 1 -p 8080:80@agent:0  # for version < 5 the nodefilter needs to be specified as "@agent[0]"
~~~

### Install the Helm charts
We need a docker image of the application that's pushed to a registry. You can use the image from our public quay.io: [quay.io/django-hurricane/spacecrafts-demo](https://quay.io/repository/django-hurricane/spacecrafts-demo){:target="_blank"}.

The Helm charts for the spacecrafts application are contained in the `helm` directory. 
They have been created with [<ins>**our cookiecutter**</ins>](https://github.com/django-hurricane/hurricane-based-helm-template){:target="_blank"} for Helm charts for a Hurricane-based Django app.

If we look in the `deployment.yaml` we can see the usage of Hurricanes `serve` command:
~~~bash
# helm/spacecrafts/templates/deployment.yaml
[...]
          args: ["python manage.py serve
                  --command 'migrate'
                  --command 'collectstatic --no-input'
                  --port {{ .Values.containerPort.portNumber }}
                  --probe-port {{ .Values.containerProbePort }}
                  {{- if .Values.serveStatic }}
                  --static
                  {{ end -}}
                  {{- if .Values.serveMedia }}
                  --media
                  {{ end -}}
                 "]
[...]

~~~

The next step is to install the dependencies and the charts. The following commands need to be run from the `helm` directory. 
If you don't already have it locally available, you may need to add the [bitnami charts](https://github.com/bitnami/charts){:target="_blank"}.
~~~bash
cd helm
helm repo add bitnami https://charts.bitnami.com/bitnami
helm dep build spacecrafts
helm install spacecrafts spacecrafts/
~~~

We can check which deployments and pods are running:
~~~bash
kubectl get deployments
kubectl get pods
~~~
This way we can inspect the logs of the spacecrafts-pod and can observe the requests from the probes:
~~~bash
kubectl logs -f spacecrafts-XXXXX-XXXXXXXX
~~~
By running this command we can get the url, which we can use to access the spacecrafts application
~~~bash
kubectl get ingress
~~~

You might want to install the fixture, which you can using following commands:
~~~bash
kubectl exec -it spacecrafts-XXXXX-XXXXXXXX -c spacecrafts -- bash
python manage.py loaddata components
~~~
If you don't load it, you're strongly advised to create a component with the title "Main engine".
The spacecrafts application also contains a custom check handler, which is described in more detail in [**this guide**](https://django-hurricane.io/custom-checks/).
A big advantage of Django Hurricane is that you don't need to write a lot of boilerplate code, i.e. probe handlers for Kubernetes probes, Hurricane takes care of it. 
So we don't have to do anything on that end here.

You should be able to access the application using the ingress hostname, prepended by the port you specified when creating the k3d cluster (in this case [**spacecrafts.dev.127.0.0.1.nip.io:8080**](http://spacecrafts.dev.127.0.0.1.nip.io:8080){:target="_blank"})
and for instance you can access the graphql background at [**spacecrafts.dev.127.0.0.1.nip.io:8080/graphql**](http://spacecrafts.dev.127.0.0.1.nip.io:8080/graphql){:target="_blank"}.
If DNS Rebinding doesn't work or isn't allowed on your local setup and therefore you can't use [**nip.io**](https://nip.io/){:target="_blank"}, you need to add an entry to your `/etc/hosts` in order to access the url:
~~~bash
# /etc/hosts
[...]
127.0.0.1       spacecrafts.dev.127.0.0.1.nip.io
[...]
~~~


### Local Kubernetes development: code hot-reloading, debugging and more

In order to comfortably further develop the spacecrafts app in the local cluster, we should absolutely have hot-reloading of the source code.

This can be done e.g. with local path mapping of k3d.

A more comfortable way to achieve this, with supported capabilities for debugging, is [<ins>**Gefyra**</ins>](https://gefyra.dev/){:target="_blank"}.

If you want maximum convenience for your developers and a supported team oriented workflow, we recommend you check out [<ins>**Unikube**</ins>](https://unikube.io/){:target="_blank"}.
