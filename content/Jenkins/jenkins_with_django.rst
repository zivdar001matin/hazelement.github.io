Setup Jenkins with Django for Continous Deployment
###################################################

:date: 2016-12-28
:tags: Jenkins, Django, Continous Deployment
:authors: Harry Zheng


About Jenkins
==================

Jenkins is a tool that is widely used for continous integration/deployment. It's basically a tool to automate the process of writing code, running tests and deploy for production.

We will be using this post to demonstrate the setup using a "hello world" Django web application. The web application will be served using apache. 

However, the setup can be used with any web development language and framework.

The Setup
=========
Our development code will be sitting at ~/django_hello_world. 

For simplicity, the source code will be pushed to a git repository that is sitting on the same machine at /webapp_repo. It can be on other machine through a ssh tunnel. 

The production code, which is also our live code, will be sitting at /webapps/django_hello_world


Django Hello World
==================

First thing first, we need to setup an web application first. This will be a similar version of the official Django tutorial, https://docs.djangoproject.com/en/1.10/intro/tutorial01/.

Start the Django project:

.. code-block:: bash

	cd ~/django_hello_world
	django-admin startproject hello_world

Under the same directory, create an app called polls:

.. code-block:: bash

	python manage.py startapp polls

This will create a folder called polls. Under polls, open up the file called views and put the following code:

.. code-block:: python
	
	from django.http import HttpResponse

	def index(request):
    	return HttpResponse("Hello, world. You're at the polls index.")

In the same polls folder, create a file named "urls.py" which will define the urls. In the urls file, enter the following code:

.. code-block:: python

	from django.conf.urls import url

	from . import views

	urlpatterns = [
	    url(r'^$', views.index, name='index'),
	]

Now let's link this url file to the global url file. Open up the urls.py file under hello world folder and put in the following code:

.. code-block:: python

	from django.conf.urls import url, include
	from django.contrib import admin

	urlpatterns = [
	    url(r'^polls/', include('polls.urls')),
	    url(r'^admin/', admin.site.urls),
	]

And that's it! Go back to project root directory:

.. code-block:: bash

	cd ~/django_hello_world

And run this command:

.. code-block:: bash

	python manage.py runserver


Open up http://localhost:8000/polls/ in the browser and we should see the text "Hello, world. You're at the polls index."

Remember this only runs our web application under local host, and only we can see it. In order for other people to see it, we need to use apache to serve it. 


Serve Django with Apache 
========================
Django's tutorial website has a thorough documentation on the setup, https://docs.djangoproject.com/en/1.10/howto/deployment/wsgi/modwsgi/. We will briefly mention it here. 

Using mod_wsgi daemon mode is the recommended way to serve our application. Assuming we have apache and mod_wsgi installed. 

First of all, let's create a directory to store our production files where Jenkins will be publishing to. 

.. code-block:: bash

	cd /
	mkdir /webapps/hello_world

Our production code will be sitting under this directory,

.. code-block:: bash

	hello_world/
		hello_world/
		polls/
		...


Under apache's enabled site directory, let's create a conf file for our web application. 

.. code-block:: bash

	cd /etc/apache2/sites-enabled
	touch hello_world.conf

In hello_world.conf, enter the following contents,

.. code-block:: bash

	Listen 8888
	<VirtualHost *:8888>
	        # The ServerName directive sets the request scheme, hostname and port that
	        # the server uses to identify itself. This is used when creating
	        # redirection URLs. In the context of virtual hosts, the ServerName
	        # specifies what hostname must appear in the request's Host: header to
	        # match this virtual host. For the default virtual host (this file) this
	        # value is not decisive as it is used as a last resort host regardless.
	        # However, you must set it for any further virtual host explicitly.
	        #ServerName www.example.com

	        # ServerAdmin webmaster@localhost
	        # DocumentRoot /var/www/html

	        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
	        # error, crit, alert, emerg.
	        # It is also possible to configure the loglevel for particular
	        # modules, e.g.
	        # LogLevel info ssl:warn


	        WSGIScriptAlias / /webapps/hello_world/hello_world/wsgi.py
	        WSGIDaemonProcess helloworld.com python-path=/django_hello_world
	        WSGIProcessGroup helloworld.com
	        <Directory> /webapps/hello_world/hello_world>
	                <Files wsgi.py>
	                Require all granted
	                </Files>
	        </Directory>

	        ErrorLog ${APACHE_LOG_DIR}/error.log
	        CustomLog ${APACHE_LOG_DIR}/access.log combined

	        # For most configuration files from conf-available/, which are
	        # enabled or disabled at a global level, it is possible to
	        # include a line for only one particular virtual host. For example the
	        # following line enables the CGI configuration for this host only
	        # after it has been globally disabled with "a2disconf".
	        #Include conf-available/serve-cgi-bin.conf
	</VirtualHost>

This document tells apache about configurations of our site. Apache will be listening on port 8888 and our site should be accessible on port 8888 at all IP addresses.
There is a wsgi.py file we need to create for apache to load the Django application. Go back to our development directory,

.. code-block:: bash

	cd ~/django_hello_world/hello_world
	cd hello_world
	touch wsgi.py

The file structure should look like this,

.. code-block:: bash

	hello_world/
		hello_world/
			wsgi.py
			...
		polls/
		...

In wsgi.py, enter the following content,

.. code-block:: python

	import os
	from django.core.wsgi import get_wsgi_application

	os.environ.setdefault("DJANGO_SETTINGS_MODULE", "hello_world.settings")
	application = get_wsgi_application()


And that's it, we are set for apache. Now let's get our code into our production directory.

Setup Git Repository
================================

Next setup is to setup the git repository and pull code to our production directory. For simplicity, we will setup the git repository as a local directory on our machine. 

.. code-block:: bash

    cd /
    mkdir git_repo
    cd git_repo
    git init --bare hello_world.git

This will setup a local git repository under /git_repo called hello_world.git. Next, let setup our development code to track this directory and push our code to it. 

.. code-block:: bash

	cd /django_hello_world/hello_world
	git remote add origin /git_repo/hello_world.git

Make our first commit by typing,

.. code-block:: bash
	git status
	git add --all
	git commit -a

Type in our first commit message and finish the commit. Next let's push our first commit to git repository,
	
.. code-block:: bash

	git push -u origin master

This will push our commit to remote called origin and setup our local master to track the remote master branch. Next let's pull our code into our production directory.

.. code-block:: bash

	cd /webapps
	git clone /git_repo/hello_world.git

Our lastest code should show up in the webapps directory. This is also where apache will be accessing our site code. Restart apache server, 

.. code-block:: bash

	sudo systemctl restart apache2

Now, we should be able to see the site under 8888 port, try localhost:8888 or 127.0.0.1:8888 in our web browser. 


Setup Jenkins to Link Everything
================================

















