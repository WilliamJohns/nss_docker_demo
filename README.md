# Live Demo Walkthrough

## Step 1 - Create the Django application
- In a working Python virtualenv, run the following
```bash
# Install the Django framework
pip install Django

# Make the Django project and change to the directory
django-admin startproject live_demo
cd live_demo

# Run Database migrations, and start up the server
python manage.py migrate
python mange.py runserver
```

At this point you should be able to visit `localhost:8000` in your web browser and see the site.

## Step 2 - Switch to a better Database
SQLite is the default database for Django, which is fine enough.

However we are going to switch to Postgres, because it is generally accepted as better

First, make a backup copy of `settings.py` as `settings_sqlite.py`. We are going to use this in a later step. 

Then update the `settings.py` file with the following changes

```python
...

from pathlib import Path
from os import getenv  # Add getenv to load DB connection args from the environment 
...
# Update DATABASES declaration with Postgres engine, env variables, and reasonable defaults

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': getenv('DB_NAME', 'demo_db'),
        'USER': getenv('DB_USER', 'demo'),
        'PASSWORD': getenv('DB_PASS', 'VerySecure'),
        'HOST': getenv('DB_HOST', 'localhost'),
        'PORT': getenv('DB_PORT', '5432'),
    }
}
...
```

Then run the following
```bash
# Install the PostgreSQL driver library for Python
pip install psycopg2-binary

# Start the web server
python manage.py runserver
```

At this point you should be met with an error, that the web app can't connect to the database.
SQLite just uses a single file to store data in, but PostgreSQL is a full blown engine.
Since we don't have postgres running on our machine, our web app can't connect


## Step 3 - Enter Docker
Behold, our first use for Docker. Service dependencies.
In this demo, it is shown with a simple database dependency, but it could be any number of things, like a 3rd party service, other microservices from different teams, a frontend if you were working on just an API, or all of the above.

We could resolve this issue by manually installing Postgres, and starting it up on our machine. Then we have another thing to manage on our machine though, and it's difficult to upgrade, or good luck to you have you need multiple versions of Postgres ( say for different applications ).

Alternatively, Postgres has many images [hosted on DockerHub](https://hub.docker.com/_/postgres), that we can pull down and run for free.

Assuming you have Docker on your machine, you can just run the folliwing
```bash
# Pull down the version of Postgres we want to use
docker pull postgres:13-alpine

# Start the database
docker run 
    --rm \
    -p 5432:5432 \
    -v $(pwd)/.volumes/database/:/var/lib/postgresql/data \
    -e POSTGRES_DB=demo_db \
    -e POSTGRES_USER=demo \
    -e POSTGRES_PASSWORD=VerySecure \
    postgres:13-alpine

# --rm Is a flag that tells Docker to remove the container when we are done
# -p Is short for --port and connects our machine to the container
# -v Is short for --volume and is means of persisting the data between runs
# -e Is short for --environment and sets certain variables for running the container, namely our
#    database name, username, and password
# Finally postgres:13-alpine is the name of the image we want to run a container of
```

You can now run
```bash
python manage.py migrate
python manage.py runserver
```
And should be able to resolve it in your web browser.

### Step 4 - Dockerize our Application
Local dependencies is one of the common uses for Docker, but next up is the packaging and distrubtion of custom software, in this case our Web App.

Before we begin, rename `settings.py` to `settings_postgres.py` and rename `settings_sqlite.py` to `settings.py`. This removes the service dependency for our database for now, and we will build that in next.

Create a new file in the directory called `Dockerfile`, which should look like the following.

```Dockerfile
# Base off of Python 3 image
FROM python:3

# Do not prompt for input
ENV PYTHONUNBUFFERED=1

# Make a new directory, in root, called "app", which is where commands will run by default
WORKDIR /app

# Install our dependencies ( it is better to do this with requirements.txt, but skipping that for the Demo)
RUN pip install Django

# Copy our source code over
COPY . .

# EXPOSE doesn't do anything, except inform other people/users that the given port is use by the container.
EXPOSE 8000
CMD [ "python", "manage.py", "runserver", "0.0.0.0:8000" ]
```

Then run the following commands
```bash
# Build the image
docker build -t demo:latest .
# -t Is short for --tag, which is just a label to identify this image:version
# The format is conventional {application_name}:{version_label}, and latest is frequently used for the most recent
# the '.' at the end, just tells Docker where to look for the Dockerfile with the build instructions.

# Run the new image in a container
docker run \
  --rm \
  -p 8000:8000 \
  demo:latest
```

You should now be able to view the Dockerized app, in your web browser.

## Step 5 - Putting it all together
We've looked at using an external service with a local app, and turing a local app into a Docker Image.

Now it's time to combine the two. This can be done with some manually networking, but it is easier to do, and a great reason to introduce a new tool; Docker-Compose

Docker-Compose is another tool, aimed at container orchastration. That is to say, configuring multiple containers to work in tandem. It has it's own depth of things to learn, but we are going to focus on our explicit use case for now.

First, rename `settings.py` to `settings_sqlite.py` and rename `settings_postgres.py` to `settings.py`. 
Create a new file in your directory titles `docker-compose.yaml`, which should contain the following.

```yaml
version: "3"

services:
  app:
    build: .
    ports:
      - 8000:8000
  database:
    image: postgres:13-alpine
    environment:
      - POSTGRES_DB=demo_db
      - POSTGRES_USER=demo
      - POSTGRES_PASSWORD=VerySecure
    ports:
      - 5432:5432
    volumes:
      - .volumes/postgres:/var/lib/postgresql/data
```

This is a docker-compose representation of the two steps we did previously, in a single file.

By default, all services in a compose file, share the same network.

You can stand everything up with the command

`docker-compose up`

However,you may notice when you do, that your web app fails to connect to the database. That's because in our `settings.py` file, it's still trying to connect to our host machine, and we need it to connect to the other container instead. This is achieved by adding

```yaml
 services:
   app:
    ...
    environment:
    - DB_HOST=database
    ...
```
to our compose file. Docker will take care of the networking to resolve `database` to the approriate service.

## Step 6 (Optional) - Sharing Images
The code for this demo is [pushed to DockerHub](https://hub.docker.com/repository/docker/wjohns/nss-demo), and you can pull it down and run it, rather than recreating it all.

Modify your compose file, replacing the `build .` line, under the `app` service, with `image: wjohns/nss-demo:latest`, and you only need the compose file to run everything!