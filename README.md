# Docker project
In this test I have set up django and postgres containers using docker

## How it works

### Pulling a python image and setting up django
```
FROM python:3

ENV PYTHONUNBUFFERED 1

ENV PG_DB_PASS password


RUN mkdir /django-docker

WORKDIR /django-docker

ADD requirements.txt /django-docker/

RUN pip install -r requirements.txt

ADD . /django-docker/

EXPOSE 8000

RUN python manage.py makemigrations 
RUN python manage.py migrate
RUN python manage.py runserver 0.0.0.0:8000




```
### Setting up postgres in an ubuntu image
```

 
FROM ubuntu:14.04

ENV PG_DB_PASS password
 
RUN apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys B97B0AFCAA1A47F044F244A07FCC7D46ACCC4CF8
 
RUN echo "deb http://apt.postgresql.org/pub/repos/apt/ precise-pgdg main" > /etc/apt/sources.list.d/pgdg.list
 
RUN apt-get update && apt-get install -y python-software-properties software-properties-common postgresql-9.3 postgresql-client-9.3 postgresql-contrib-9.3

 
USER postgres
 
RUN    /etc/init.d/postgresql start &&\
    psql --command "CREATE USER daniel WITH SUPERUSER PASSWORD '$PG_DB_PASS';" &&\
    createdb -O daniel my_database
 
RUN echo "host all  all    0.0.0.0/0  md5" >> /etc/postgresql/9.3/main/pg_hba.conf
 

RUN echo "listen_addresses='*'" >> /etc/postgresql/9.3/main/postgresql.conf
 
# Expose the PostgreSQL port
EXPOSE 5432
 
# Add VOLUMEs to allow backup of config, logs and databases
VOLUME  ["/etc/postgresql", "/var/log/postgresql", "/var/lib/postgresql"]
 
# Set the default command to run when starting the container
CMD ["/usr/lib/postgresql/9.3/bin/postgres", "-D", "/var/lib/postgresql/9.3/main", "-c", "config_file=/etc/postgresql/9.3/main/postgresql.conf"]
```

### Building the containers using docker compose
```
version: '3'

services:
  web:
    container_name: django-app
    build:
      context: .
      dockerfile: Dockerfile-django
    volumes:
      - .:/django-docker
    restart: always
    ports:
      - "8000:8000"
    links:
      - postgresdb
  postgresdb:
    container_name: postgresdb
    build:
      context: .
      dockerfile: Dockerfile-postgres
    restart: always
    ports:
      - '5432:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    environment:
      - PG_DB_PASS=password

volumes:
  postgres_data:

```

### Accessing the postgres database in our django project 
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'my_database',
        'USER':'daniel',
        'HOST':'postgresdb',
        'PASSWORD':os.environ['PG_DB_PASS'],
        'PORT':5432
    }
}

```
