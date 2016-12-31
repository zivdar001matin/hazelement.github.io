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
	mkdir /webapps/hello_world_app

Our production code will be sitting under this directory,

.. code-block:: bash

	hello_world_app/
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


	        WSGIScriptAlias / /webapps/hello_world_app/hello_world/wsgi.py
	        WSGIDaemonProcess helloworld.com python-path=/django_hello_world
	        WSGIProcessGroup helloworld.com
	        <Directory> /webapps/hello_world_app/hello_world>
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


And that's it, we are set for apache. Finally, restart our apache server, 

.. code-block:: bash

	sudo systemctl restart apache2

Now, we should be able to see the site under 8888 port, try localhost:8888 or 127.0.0.1:8888 in our web browser. 


Setup Jenkins to Link Everything
================================








.. _ghp-import: https://github.com/davisp/ghp-import

The idea is basically to push Pelican's output folder to the Github repository we just created. To achive this, we have a great tool called ghp-import_. We can install it easily with the following command:

.. code-block:: bash

	pip install ghp-import

A normal command using ghp-import is like follows:

.. code-block:: bash

	pelican content -o output -s pelicanconf.py
	ghp-import output
	git push git@github.com:username/username.github.io.git gh-pages:master

The first line generate all rst documents into our output folder. ghp-import then import this output folder into a git branch called gh-pages. 

Then the last line push this branch to its remote repository which is our Github User Pages repo, yourname.github.io. gh-pages is our local branch and master is our remote repository branch. Please note that the remote branch must be the master branch for it to work properly.

The last command will prompt your enter your username and password. Simply enter that, our output folder should be pushed to the repo successfully. Now open up browser and enter username.github.io and we can see our blog live on the web.

Github 2-step Authentification
==============================

If you have 2-step Authentification setup on Github account, using the git push command might not work for you as it doesn't implement a way for you to enter the second passcode. 

There are 2 ways to solve this problem, one way is to generate a app hash string passcode from Github and use it while doing git push. It's annoying as we have to keep that somewhere and we need to enter that everytime we perform a push. 

Another easier way is to generate a ssh and put your public key to github. 

Github SSH Access
=====================
There are many tutorials on how to generate a SSH key pair. A SSH key pair contains a private key and a public key. 

A private key is our personal key and we should alway keep secret. A public key is like a lock that matches our private key. When we need to access some machine remotely, we give public key to remote mahine manager and he can install it to his machine. It's like installing our lock onto his house's front door (in this case, his house would be the machine). So we can get into his house with our prviate key. And of course, he can setup locks that comes from other people, ie his door can have many locks and each lock can open his door. 

If you are using a linux machine, a quick way to generate a key pair is to use ssh-keygen. 

.. code-block:: bash

	ssh-keygen -t rsa -C "your_email@example.com"

This will generate two files, id_rsa, and id_rsa.pub. Open id_rsa.pub with any text editor and copy its entire content. 

Now let's put this public key to our Github account so that we can use our private key to access Github. 

In https://github.com/settings/keys, we have a section to add new public key. Paste everything from public key into "Key" section and name "title" to "mySSHKey". And click on add SSH key. We should be good to go. 

Next time when we do a git push, it should stop asking us about our password. 

Streamline With A Script
========================

We can stream line the publishing process with a bash script. Let's first go to our project root folder and create a file called pubish. 

.. code-block:: bash

	touch publish

In this file, let's enter the following content

.. code-block:: bash

	#!/bin/bash
	pelican content -o output -s pelicanconf.py
	ghp-import output
	git push git@github.com:username/username.github.io.git gh-pages:master

Making this file executable:

.. code-block:: bash

	chmod +x publish

We just created a script to perform the publishing process for us. To publish new content, simply enter this command under project root directory.

.. code-block:: bash

	./publish

It should perform all the task for us.











