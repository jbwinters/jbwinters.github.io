---
layout: post
title:  "Nginx/gunicorn/flask in Docker"
date:   2016-01-15 16:27:58
---
Spent some time setting up a flask server on top of gunicorn/nginx/supervisor in Docker. I didn't find a comprehensive resource on how to do this the way I wanted. Specifically, I wanted Werkzeug-like WSGI logging which gunicorn doesn't do on its own, and logging from the app streamed to stdout which gunicorn and supervisor both disable by default. Fortunately this task ended up being fairly simple!

Here's the directory structure:

	myapp/
		app.conf
		Dockerfile
		requirements.txt
		run.py
		supervisord.conf

Dockerfile:

	from python:2.7

	# Install system requirements
	RUN apt-get update
	RUN apt-get install -y gunicorn nginx supervisor

	# Set up app
	ENV APPDIR /usr/src/app
	RUN mkdir -p ${APPDIR}
	WORKDIR ${APPDIR}
	COPY requirements.txt ${APPDIR}
	RUN pip install --no-cache-dir -r requirements.txt
	COPY . ${APPDIR}

	# Set up nginx
	ENV APPCONF app.conf
	RUN rm /etc/nginx/sites-enabled/default
	COPY ${APPCONF} /etc/nginx/sites-available/
	RUN ln -s /etc/nginx/sites-available/${APPCONF} /etc/nginx/sites-enabled/${APPCONF}
	RUN echo "daemon off;" >> /etc/nginx/nginx.conf

	# Set up supervisord
	RUN mkdir -p /var/log/supervisor
	COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

	# Expose web port
	EXPOSE 80

	# Start processes
	CMD [ "/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf" ]

app.conf (the nginx site config):
{% highlight nginx %}
server {
    listen      80;

    location / {
        proxy_pass http://localhost:5000/;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

{% endhighlight %}

supervisord.conf:

	[supervisord]
	nodaemon=true

	[program:nginx]
	command=/usr/sbin/nginx
	stdout_logfile=/dev/stdout
	stdout_logfile_maxbytes=0
	stderr_logfile=/dev/stderr
	stderr_logfile_maxbytes=0

	[program:gunicorn]
	directory=/usr/src/app
	command=gunicorn -b localhost:5000 --enable-stdio-inheritance run:loggingapp
	environment=PYTHONUNBUFFERED=true
	stdout_logfile=/dev/stdout
	stdout_logfile_maxbytes=0
	stderr_logfile=/dev/stderr
	stderr_logfile_maxbytes=0
	accesslog=-

The stdout_logfile/stderr_logfile stuff makes `[program:x]` output visible when supervisord is not a daemon. gunicorn is launched with `--enable-stdio-inheritance` so output is properly redirected. `run:loggingapp` tells gunicorn to run the `loggingapp` app in the `run` module.

requirements.txt:

	flask==0.10.1
	itsdangerous==0.24
	Jinja2==2.8
	MarkupSafe==0.23
	Werkzeug==0.11.3
	wheel==0.24.0
	wsgi-request-logger==0.4.4

Bump versions as needed. For this example you only need to install flask and wsgi-request-logger. We use [wsgi-request-logger](https://github.com/pklaus/wsgi-request-logger) to do the real work of logging WSGI events.

run.py:
{% highlight python %}
import logging

import flask
from requestlogger import WSGILogger, ApacheFormatter

app = flask.Flask(__name__)

loggingapp = WSGILogger(app, [logging.StreamHandler()], ApacheFormatter())

if __name__ == '__main__':
    # This is only used when calling `python run.py` directly! Useful if you want to use flask's debug mode.
    app.run(host='0.0.0.0')  
{% endhighlight %}

It doesn't matter how you get `app` into run.py here, you could import it from some other module. However, if you don't want to run the app from run.py you'll have to change the gunicorn launch command in supervisor.conf to reference your desired module instead of `run`. Note that `ApacheFormatter` is wsgi-request-logger's default formatter and can be replaced with a custom formatter as desired.

Nginx passes web traffic (port 80) through to gunicorn's exposed port 5000. We expose the container's port 80:

	docker build -t server -f Dockerfile
	docker run -p 80:80 server

After all this is set up and the docker container is running, curl your endpoint (hosted at the docker host's port 80). The WSGI request will be posted to your container's log (stdout with the above docker commands). In fact, all of your app's logging events will be posted!
