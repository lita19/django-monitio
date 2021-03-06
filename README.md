Monitio for Django
==================

Monitio allows you to have messages (aka notifications), that:

* can be persisted (stored in the database and read later),
* which dynamically show on the web UI when added,
* and can be optionally sent via e-mail to your end-user.

Monitio is built upon:

* [django-sse](https://github.com/niwibe/django-sse)
 * which uses [django-redis](https://github.com/niwibe/django-redis)
 * ... and [Redis database](https://redis.io)
* [Yaffle's EventSource.js](https://github.com/Yaffle/EventSource) for cross-browser Server-Sent Events compatibility
* [django-cors-headers](https://github.com/ottoyiu/django-cors-headers) for the same thing
* [django-transaction-signals](https://github.com/davehughes/django-transaction-signals)
* [jQuery](http://jquery.com/) and [jQueryUI](http://jqueryui.com)

With such sophisticated setup, using packages from many individuals, the demo 
application is currently properly running on MSIE 10, Opera 12, FFox 16 and 
Safari 5.1.7 on Windows. Also, monitio has been tested in production environment
with nginx + gunicorn, which also has been found to work properly. 


Long explanation
----------------
There are situations in Django, when you want to perform a task in background,
via Celery or something similar, then you want to inform the user about it.

The process may be just a few seconds long and the user may have not yet
closed the browser window or changed the web page. 

Or, the process may take much more time. 

The user may already have closed the browser window, right? 

So let's inform the user after he/she logs in again. And, let's have an option to give
him or her some information over e-mail. That would be cool.

On the other hand, what if the user waits patiently, staring at the browser window? Let's give him or her a dynamic pop-up with a message. Let's do that in a cool way, using HTML5's [SSE](http://en.wikipedia.org/wiki/Server-sent_events)

This project is an attept to be a definitive turnkey solution for such situations.
It is built on [maurojp](https://github.com/maurojp) fork of django-persistent-messages by [philomat](https://github.com/philomat),
 it uses django-sse with Redis for dynamic notification and EventSource as a polyfill for older browsers, that don't support SSE.

So, as a result of using this piece of software, you get dynamic, lightweight
notifications for your end-users, INCLUDING anonymous users, with an option to
leave persistent messages. And e-mail your peeps. Isn't that cool?

Documentation
-------------

A Django app for unified, persistent and live user messages/notifications, built on top of Django's [messages framework](http://docs.djangoproject.com/en/dev/ref/contrib/messages/) (`django.contrib.messages`).

Monitio is a messages storage backend that provides support for messages that are supposed to be persistent, that is, they outlast a browser session and will be stored in the database. These messages can be displayed as you will to the user, you can let the user mark them as read, remove them or even reply them. For some of these actions there are views you can import in your project urls.py.

* Support persistent and nonpersistent messages for authenticated users. Persistent messages are stored in the database. 
* For anonymous users, messages are stored using the cookie/session-based approach. There is no support for persistent messages for anonymous users.
* There is a unified API for displaying messages to both types of users, that is, you can use the same code you'd be using with Django's messaging framework in order to add and display messages, but there is additional functionality available if the user is authenticated.

Installation
------------

This document assumes that you are familiar with Python and Django.

1. Clone this git repository (no PyPI package for this fork). master branch is the lastest stable branch: 

        $ git clone git://github.com/mpasternak/django-monitio.git

2. Make sure `monitio` is in your `PYTHONPATH`.
3. Add `monitio` & company to your `INSTALLED_APPS` setting.

        INSTALLED_APPS = (
            ...
            'django_sse',
            'corsheaders',
            'monitio',
        )

4. Make sure Django's `MessageMiddleware` is in your `MIDDLEWARE_CLASSES` setting (which is the case by default), also enable `CorsMiddleware` there:

        MIDDLEWARE_CLASSES = (
            ...
            'django.contrib.messages.middleware.MessageMiddleware',
    		'corsheaders.middleware.CorsMiddleware',
        )

5. Add the CONTEXT_PROCESSOR for messages:

        CONTEXT_PROCESSORS = (
            ...
            'django.contrib.messages.context_processors.messages',
            ...
        )

 
6. Add the `monitio` URLs to your URL conf. For instance, in order to make messages available under `http://domain.com/messages/`, add the following line to `urls.py`.

        urlpatterns = patterns('',
            (r'^messages/', include('monitio.urls')),
            ...
        )

7. In your settings, set the message [storage backend](http://docs.djangoproject.com/en/dev/ref/contrib/messages/#message-storage-backends) to `monitio.storage.PersistentMessageStorage`:

        MESSAGE_STORAGE = 'monitio.storage.PersistentMessageStorage'
        
8. In your settings, add a reasonable default, which will prevent from showing read messages to the users:

        MONITIO_EXCLUDE_READ = True
        
9. Setup `django-sse` and `corsheaders`:

        REDIS_SSEQUEUE_CONNECTION_SETTINGS = {
            'location': '127.0.0.1:6379',
            'db': 0,
        }
        
        CORS_ORIGIN_WHITELIST = (
            '127.0.0.1',
            '127.0.0.1:8000',
        )

10. Set up the database tables using

	    $ manage.py syncdb

11. If you want to use the bundled templates, add the `templates` directory to your `TEMPLATE_DIRS` setting:

        TEMPLATE_DIRS = (
            ...
            'path/to/monitio/templates')
        )
        
12. Setup a production server - for [nginx](http://nginx.org/) + [gunicorn](http://gunicorn.org), please use configuration below:

		
		location ~ ^/messages {
			proxy_pass http://your-gunicorn-address.../
			proxy_buffering off;
			proxy_cache off;
			proxy_set_header Host $host;
			
			proxy_set_header Connection '';
			proxy_http_version 1.1;
			chunked_transfer_encoding off;
		}

   And, for [gunicorn](http://gunicorn.org), make sure you install [gevent](http://www.gevent.org/) and run [gunicorn](http://gunicorn.org) with parameter `-k gevent`


Using messages in views and templates
-------------------------------------

### Message levels ###

Django's messages framework provides a number of [message levels](http://docs.djangoproject.com/en/dev/ref/contrib/messages/#message-levels) for various purposes such as success messages, warnings etc. 

    import monitio
    # persistent message levels:
    monitio.INFO
    monitio.SUCCESS
    monitio.WARNING
    monitio.ERROR
    
This app provides constants with the same names, the difference being that messages with these levels are going to be persistent:

    from django.contrib import messages
    # temporary message levels:
    messages.INFO 
    messages.SUCCESS 
    messages.WARNING
    messages.ERROR

**Note**: Let's stress the importance of this. If you use `monitio` constants the message will be stored in the database and kept there till somebody explicitly deletes it. If you use `contrib.messages` constants, you get the same behavior as if you were using a non persistent storage, messages are stored in the database ensuring reception but they are removed right after being accessed.

### Adding a message ###

Since the app is implemented as a [storage backend](http://docs.djangoproject.com/en/dev/ref/contrib/messages/#message-storage-backends) for Django's [messages framework](http://docs.djangoproject.com/en/dev/ref/contrib/messages/), you can still use the regular Django API to add a message:

    from django.contrib import messages
    messages.add_message(request, messages.INFO, 'Hello world.')

This is compatible and equivalent to using the API provided by `monitio`:

    import monitio
    from django.contrib import messages
    monitio.add_message(request, messages.INFO, 'Hello world.')

In order to add a persistent message (one that is stored permanently in the Database), use `monitio` levels listed above:

    messages.add_message(request, monitio.WARNING, 'This message is stored in monitio table till removed.')

or the equivalent:

    monitio.add_message(request, monitio.WARNING, 'This message is stored in monitio table till removed')
    
Note that this is only possible for logged-in users, so you are probably going to have make sure that the current user is not anonymous using `request.user.is_authenticated()`. Adding a persistent message for anonymous users raises a `NotImplementedError`.

### Extended API ###

Persistent Messages has an extended API that will let you do some extra nice things. This is the prototype of `add_message` in contrib messages:

    def add_message(request, level, message, extra_tags='', fail_silently=False):

This is the prototype of `add_message` in Persistent Messages.

    def add_message(request, level, message, extra_tags='', fail_silently=False, subject='', user=None, email=False, from_user=None, expires=None, close_timeout=None):

#### Subject and email notifications ####

Using `monitio.add_message`, you can also add a subject line to the message. You can also set if you want an email notification to be sent. The following message will be stored as a message in the database and also sent to the email address associated with the current user:

    monitio.add_message(request, monitio.INFO, 'Message body', subject='Please read me', email=True)

**Note!** Email notifications at the moment are too simple, I don't recommend using them, I'm not.

#### Send messages to different users ####

You can also pass this function a `User` object if the message is supposed to be sent to a user other than the one who is currently authenticated. User Sally will see this message the next time she logs in:

    from django.contrib.auth.models import User
    sally = User.objects.get(username='Sally')
    monitio.add_message(request, monitio.SUCCESS, 'Hi Sally, here is a message to you.', subject='Success message', user=sally)
    
You can also set a `from_user`, which lets you use Persistent Messages as messaging system between users.

#### You can make messages expire ####

You need to pass a date and time to `expires` argument. Once the message has expired, it will not be included in the returned QuerySet. At the moment there is no view or method to clear expired messages from database.

### Displaying messages ###

Messages can be displayed [as described in the Django manual](http://docs.djangoproject.com/en/dev/ref/contrib/messages/#displaying-messages). However, you are probably going to want to include links tags for closing each message (i.e. marking it as read). In your template, use something like:

    {% if messages %}
    <ul class="messages">
        {% for message in messages %}
        <li{% if message.tags %} class="{{ message.tags }}"{% endif %}>
            {% if message.subject %}<strong>{{ message.subject }}</strong><br />{% endif %}
            {{ message.message }}<br />
            {% if message.is_persistent %}<a href="{% url message_mark_read message.pk %}">close</a>{% endif %}
        </li>
        {% endfor %}
    </ul>
    {% endif %}

You can also use the bundled templates instead. The following line replaces the code above. It allows the user to remove messages and mark them as read using Ajax requests, provided your HTML page includes JQuery:

    {% include "monitio/message/includes/messages.jquery.html" %}

### Creating notifications from background tasks (eg. Celery) ###

To create a notofication from a long-running, background process, use
api.create_message:

    def create_message(to_user, level, message, from_user=None, extra_tags='',
                   subject='', expires=None, close_timeout=None, sse=True,
                   email=False):

This function will call PersistentStorage.add method for you.

### Storage extra methods ###

In Django `request._messages` is set to the default storage you configured in your settings. Persistent Messages storage has some extra methods that Django built-in storages don't have that can be very useful:

* **get_persistent**: Get read and unread persistent messages
* **get_persistent_unread**: Get unread persistent messages
* **get_nonpersistent**: Gets nonpersistent messages
* **count_unread**: Counts persistent and nonpersistent unread messages
* **count_persistent_unread**: Counts persistent unread messages
* **count_nonpersistent**: Counts nonpersistent messages

Let's see some examples of what this means.

#### Display only nonpersistent messages ####

This is reduced version of a template that would let you iterate over nonpersistent messages:

    {% if messages.get_nonpersistent %}
        {% for message in messages.get_nonpersistent %}
            [...]
        {% endfor %}
    {% endif %}

#### Display number of unread messages ####

Imagine you've created an inbox for your users using Persistent Messages and you want to show them in the menu how many unread messages they have, if they have them:

    <ul id="menu">
        <li><a href="">inbox {% if messages.count_persistent_unread > 0 %}({{ messages.count_persistent_unread }}){% endif %}</a></li>
    </ul>

### URLs and Persistent Messages Views ####

As said before you can import Persistent Messages URLs in your project's URL conf. This are the named urls you get:

* `{% url message_detail message_id %}` This renders template `monitio/message/detail.html` with specific message in the context as `message`.
* `{% url message_mark_read message_id %}` Marks specific message as read
* `{% url message_mark_all_read %}` Marks all messages of the currently logged in user as read
* `{% url message_delete message_id %}` Deletes specific message
* `{% url message_delete_all %}` Deletes all the messages of the currently logged in user 

There is plenty of room for improvement in these views and urls.

### AUTHORS ###

[philomat](https://github.com/philomat) is the author of original code for
[django-persistent-messages](https://github.com/philomat/django-persistent-messages),
which was then forked by [maurojp](https://github.com/maurojp), then it was
forked by [mpasternak](https://github.com/mpasternak).




