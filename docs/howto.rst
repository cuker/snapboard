.. _howto:

===============
SNAPboard HOWTO
===============

This HOWTO is intended to guide you through the installation and configuration
of SNAPboard as part of a Django project. It is assumed that you are familiar
with `Django`__ and have Django 1.0 installed—or that you have a compatible 
Subversion checkout on your PYTHONPATH.

__ http://www.djangoproject.com/

Get the dependencies
====================

django-pagination
-----------------

SNAPboard depends on `django-pagination`__. To install django-pagination, 
download :file:`django-pagination-1.0.4.tar.gz` (or any later version), then
run the following commands:

__ http://code.google.com/p/django-pagination/

.. code-block:: bash

    tar xvf django-pagination-1.0.4.tar.gz
    cd django-pagination
    python setup.py install

django-notification
-------------------

`django-notification`__ is not required for SNAPboard to run, but if it is not
installed, your users will not receive email notifications—for example, when
one of their watched threads is updated.

__ http://code.google.com/p/django-notification/

To install django-notification you will need to fetch the source code from
their subversion repository and place the package on your Python interpreter's
search path. This is explained in the next section.

django-mailer
-------------

`django-mailer`__ is recommended by django-notification. It provides an
asynchronous email sending facility. When django-notification—and possibly
other applications—call ``mailer.send_mail()``, the email is usually not sent
immediately; instead, it is queued and you must run ``manage.py send_mail`` for
any queued email to be sent to your SMTP server. Of course, ``manage.py
send_mail`` should be scheduled to run regularly. On UNIX systems, this would
be achieved with cron.

__ http://code.google.com/p/django-mailer/

You should read django-mailer's documentation for more information. As for 
django-notification, django-mailer must currently be installed from
the sources in the project's subversion repository.

.. note::

    If django-mailer is installed on your system, django-notification will
    attempt to use it, even if it is not configured properly.

Get SNAPboard
=============

Currently, SNAPboard is only available as a Subversion repository. Work is in
progress to make standard Python packages available. To install SNAPboard,
create a directory somewhere on your disk. Open a shell and ``cd`` into that
directory, then run:

.. code-block:: bash

    svn checkout http://snapboard.googlecode.com/svn/trunk/ .

.. note::

    The ``svn`` command is only available if you have the `Subversion 
    source control management system`__ installed. 

    __ http://subversion.tigris.org/

For Python to find the snapboard package that is inside the directory 
where you loaded the SNAPboard development trunk, that directory needs to be on
Python's search path. One way is to make the `PYTHONPATH` environment variable
point to the directory. Another way, which is available on UNIX
systems, is to make a symbolic link from a standard Python module directory to 
the snapboard package. For example, you can run 
``ln -s ~/snapboard-trunk/snapboard /usr/lib/python2.5/site-packages/``. This 
will typically require root privileges.

To test that snapboard is on your path, launch a python shell and run ``import 
snapboard``.

Tune your project's settings
============================

If you don't have a Django project where to install SNAPboard, create one 
with ``django-admin.py startproject <project name>``. Then, open 
:file:`settings.py` in your favorite text editor. Ensure the main Django
settings, such as database connection information, are set to the required
values.

.. note::

    SNAPboard should work with all SQL database backends supported by Django. 
    If you are testing SNAPboard, we recommend that you use SQLite because it is
    fast and easy to use in such setting.

Then, add ``snapboard`` and ``pagination`` to the `INSTALLED_APPS` setting::

    INSTALLED_APPS = (
        'django.contrib.auth',
        ...
        'pagination',
        'snapboard',
    )

If you use django-notification, add ``notification`` and, if django-mailer is
installed, add ``mailer``::

    INSTALLED_APPS = (
        'django.contrib.auth',
        ...
        'pagination',
        'notification',
        'mailer',
        'snapboard',
    )

Edit the `TEMPLATE_CONTEXT_PROCESSORS` setting to add 
``django.core.context_processors.request`` and 
``snapboard.views.snapboard_default_context``::

    TEMPLATE_CONTEXT_PROCESSORS = (
        "django.core.context_processors.auth",
        "django.core.context_processors.debug",
        "django.core.context_processors.i18n",
        "django.core.context_processors.media",
        "django.core.context_processors.request",
        "snapboard.views.snapboard_default_context",
    )

`TEMPLATE_CONTEXT_PROCESSORS` is not in :file:`settings.py` by default, 
so you may need to add it. The listing above has all of Django's default
context processors followed by the two required by SNAPboard.

`MIDDLEWARE_CLASSES` needs to contain, in addition to Django's defaults, two 
mandatory middleware classes and two optional ones::

    MIDDLEWARE_CLASSES = (
        ...
        "pagination.middleware.PaginationMiddleware",
        "snapboard.middleware.threadlocals.ThreadLocals",

        # These are optional
        "snapboard.middleware.ban.IPBanMiddleware",
        "snapboard.middleware.ban.UserBanMiddleware",
    )

SNAPboard also defines some setting variables that you need to insert in
:file:`settings.py`::

    # Defaults to MEDIA_URL + 'snapboard/'
    SNAP_MEDIA_PREFIX = '/media'

    # Set to False if your templates include the SNAPboard login form
    USE_SNAPBOARD_LOGIN_FORM = True

    # Select your filter, the default is Markdown
    # Possible values: 'bbcode', 'markdown', 'textile'
    SNAP_POST_FILTER = 'bbcode'

`SNAP_MEDIA_PREFIX` points to the root URL of SNAPboard's media files. This 
is needed to point the templates to the location of the required JavaScript 
files.

`USE_SNAPBOARD_LOGIN_FORM` determines whether the templates should display 
a login form. This is useful assuming you make SNAPboard inherit a custom 
base template which already has a login form: just set it to `False`.

`SNAP_POST_FILTER` indicates the formatting language your users can 
use on the forums. You should not change this setting after your forum has 
been receiving posts as existing messages would be rendered incorrectly.
If in doubt, choose 'bbcode'. SNAPboard comes with an edition toolbar to 
make BBcode easy to use for your users. It is also widely adoped.

Finally, don't forget some of Django's optional settings such as
`LOGIN_REDIRECT_URL`, `EMAIL_HOST`, etc.

Add SNAPboard to your `urlconf`
===============================

Open your project's root `urlconf`, :file:`urls.py`. At a minimum, you 
need to ``include('snapboard.urls')`` under a prefix of your choice. If you
want to use SNAPboard's login and logout templates and enable the
administration interface, you can re-use the following code::

    from django.conf.urls.defaults import *
    from django.contrib import admin
    from django.contrib.auth import views as auth_views

    admin.autodiscover()

    urlpatterns = patterns('',
        (r'^snapboard/', include('snapboard.urls')),
        (r'^accounts/login/$', auth_views.login, 
            {'template_name': 'snapboard/signin.html'}, 'auth_login'),
        (r'^accounts/logout/$', auth_views.logout, 
            {'template_name': 'snapboard/signout.html'}, 'auth_logout'),
        (r'^admin/(.*)', admin.site.root),
    )

If you use django-notification, you need to add the following under the initial
definition of `urlpatterns`::

    urlpatterns += patterns('',
        # As long as we don't include django-notification's urlconf, we must define the URL for 
        # 'notification_notices' ourselves because of notification/models.py:251.
        (r'^notices/', 'django.views.generic.simple.redirect_to', {'url': '/snapboard/'}, 'notification_notices'),
    )

SNAPboard doesn't yet have an online version of the notices sent
to users and a bug in django-notification causes an error when the named url
'notification_notices' doesn't exist. To work around that we create a fake URL
handler that just redirects to our forums. The issue is likely to be fixed in 
a future release of django-notification.

In a development setting, you may also want to serve the media files via
Django's integrated web server. To do so, add::

    from django.conf import settings
    if settings.DEBUG:
        urlpatterns += patterns('',
            (r'^media/(?P<path>.*)$', 'django.views.static.serve', 
                {'document_root': settings.MEDIA_ROOT}),
        )

You'll also need to set `MEDIA_ROOT` in :file:`settings.py`.

.. admonition:: Warning

    Do not use 'django.views.static.serve' outside of a development
    environment. In production, have your web server serve your media files 
    statically. This is both more efficient and more secure.

Done !
======

SNAPboard is set up, all that is left is to run ``./manage.py syncdb`` from
within your project directory. If `settings.DEBUG` is true, SNAPboard will
offer to install some sample data. If you are trying out SNAPboard for the 
first time, you should probably accept.
    
.. code-block:: python

        You've installed SNAPboard with DEBUG=True, do you want to populate
        the board with random users/threads/posts to test-drive the
        application?
            (yes/no):
            yes
        thread  0 created
        thread  1 created
        thread  2 created
        thread  3 created
        thread  4 created
        ...

Getting help
============

If you need help with this tutorial or want to discuss SNAPboard, use our 
mailing list `snapboard-discuss@googlegroups.com`. To register or consult the 
archives, check out http://groups.google.com/group/snapboard-discuss.

