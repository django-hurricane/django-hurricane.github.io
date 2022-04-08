---
layout: page
title: Guide to add a custom check
---
## What you'll learn
- Add a custom check to an existing project
- Run it in local Kubernetes with k3d

## What you'll need
- [poetry](https://python-poetry.org/docs#installation){:target="_blank"}
- [k3d](https://k3d.io/v5.4.1/#installation){:target="_blank"}
- [helm](https://helm.sh/docs/intro/install/){:target="_blank"}
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl){:target="_blank"}

## Writing and configuring a custom check

We're building on top of the spacecrafts app, which is described in more detail in [**this guide**](https://django-hurricane.io/basic-app/){:target="_blank"}.
You need to have it running locally in order to follow this guide.

The custom check handler is in a file named `src/apps/components/checks.py` and looks like follows:

~~~python
# src/apps/components/checks.py
import logging

from django.core.checks import Error

from apps.components.models import Component

logger = logging.getLogger("hurricane")


def example_check_main_engine(app_configs=None, **kwargs):
    """
    Check for existence of the "Main engine" component in the database
    """

    # your check logic here
    errors = []
    logger.info("Our check has been called :]")
    # we need to wrap all sync calls to the database into a sync_to_async wrapper for Hurricane to use it in async way
    if not Component.objects.filter(title="Main engine").exists():
        errors.append(
            Error(
                "an error",
                hint="There is no main engine in the spacecraft, it need's to exist with the name 'Main engine'. "
                "Please create it in the admin or by installing the fixture.",
                id="components.E001",
            )
        )

    return errors
~~~

You can see that it is a very simple check, that checks whether a component with the title "Main engine" exists.
Because how can you fly a spacecraft without a main engine, right? If it isn't found, the check returns an error.

Next, we set a default app config in `apps/components/__init__.py`

~~~python
# src/apps/components/__init__.py
default_app_config = 'apps.components.apps.ComponentsConfig'
~~~

Now we need to register this check, so that Django can use it in its check logic. 
Note that we register the check with the tag `hurricane`, as Hurricane only runs check with that tag.
Your `apps/components/apps.py` should have following content:

~~~python
# apps/components/apps.py
from django.apps import AppConfig


class ComponentsConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "apps.components"

    def ready(self):
        from django.core.checks import register

        from apps.components.checks import example_check_main_engine

        register(example_check_main_engine, "hurricane")
~~~

We register our check only after the application is ready, otherwise we will run into the error of `AppNotReady`.
This way we make sure, that this check is only registered after the application is ready, as it requires a connection
to the model. 

To verify whether the check works, you can just delete the `Main engine` component from the database.
If you're running Hurricane in a local virtualenv, you can also directly browse to `/alive` at the probe port to inspect its output. See the [**basic spacecrafts guide**](https://django-hurricane.io/basic-app/){:target="_blank"} for how to do that. 
