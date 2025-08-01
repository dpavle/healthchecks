# Self-Hosted Healthchecks

Healthchecks is open-source, and is licensed under the BSD 3-clause license.

As an alternative to using the hosted service at
[https://healthchecks.io](https://healthchecks.io), you have the option to host a
Healthchecks instance yourself.

The building blocks are:

* Python 3.10+
* Django 5.1
* PostgreSQL or MySQL

## Setting Up for Development


You can set up a development environment in a Python
[virtual environment](https://docs.python.org/3/tutorial/venv.html)
on your local system to develop a new feature, write a new integration
or test a bugfix.

The following instructions assume you are using a Debian-based OS.

* Install dependencies:

        $ sudo apt-get update
        $ sudo apt-get install -y gcc python3-dev python3-venv

* Prepare directory for project code and virtualenv. Feel free to use a
  different location:

        $ mkdir -p ~/webapps
        $ cd ~/webapps

* Prepare virtual environment
  (with virtualenv you get pip, we'll use it soon to install requirements):

        $ python3 -m venv hc-venv
        $ source hc-venv/bin/activate

* Check out project code:

        $ git clone https://github.com/healthchecks/healthchecks.git

* Install requirements (Django, ...) into virtualenv:

        $ pip install wheel
        $ pip install -r healthchecks/requirements.txt


* Create database tables and a superuser account:

        $ cd ~/webapps/healthchecks
        $ ./manage.py migrate
        $ ./manage.py createsuperuser

    With the default configuration, Healthchecks stores data in a SQLite file
    `hc.sqlite` in the project directory (`~/webapps/healthchecks/`).

* Run tests:

        $ ./manage.py test

* Run development server:

        $ ./manage.py runserver

* From another shell, run the `sendalerts` management command, responsible for
  sending out notifications:
        $ ./manage.py sendalerts

At this point, the site should now be running at `http://localhost:8000`.

## Accessing Administration Panel

Healthchecks comes with Django's administration panel where you can perform
administrative tasks: delete user accounts, change passwords, increase limits for
specific users, inspect contents of database tables.

To access the administration panel, if you haven't already, create a superuser account:

    $ ./manage.py createsuperuser

This will trigger an interactive prompt.

You can also provide credentials via parameters, bypassing the interactive prompt:

    $ ./manage.py createsuperuser --email user@example.com --password changeme123

Then, log into the site using the superuser credentials. Once logged in,
click on the "Account" dropdown in top navigation, and select "Site Administration".

## Sending Emails

Healthchecks needs SMTP credentials to be able to send emails:
login links, monitoring notifications, monthly reports.

Specify SMTP credentials using the `EMAIL_HOST`, `EMAIL_PORT`, `EMAIL_HOST_USER`,
`EMAIL_HOST_PASSWORD`, `EMAIL_USE_SSL`, and `EMAIL_USE_TLS` environment variables.
Example:

```ini
EMAIL_HOST=my-smtp-server-here.com
EMAIL_PORT=465
EMAIL_HOST_USER=my-username
EMAIL_HOST_PASSWORD=mypassword
EMAIL_USE_SSL = True
EMAIL_USE_TLS = False
```

You can read more about handling outbound email in the Django documentation,
[Sending Email](https://docs.djangoproject.com/en/4.2/topics/email/) section.

## Receiving Emails

Healthchecks comes with a `smtpd` management command, which starts up an
SMTP listener service. With the command running, you can ping your
checks by sending email messages.

Start the SMTP listener on port 2525:

    $ ./manage.py smtpd --port 2525

Send a test email:

    $ curl --url 'smtp://127.0.0.1:2525' \
        --mail-from 'foo@example.org' \
        --mail-rcpt '11111111-1111-1111-1111-111111111111@my-hc.example.org' \
        -F '='

## Sending Status Notifications

The `sendalerts` management command continuously polls the database for any checks
changing state, and sends out notifications as needed.
When `sendalerts` is not running, the Healthchecks instance will not send out any
alerts.

Within an activated virtualenv, run the `sendalerts` command like so:

    $ ./manage.py sendalerts


In a production setup, make sure the `sendalerts` command can survive
server restarts.

## Database Cleanup {: #database-cleanup }

Healthchecks deletes old entries from `api_ping`, `api_flip`, and `api_notification`
tables automatically. By default, Healthchecks keeps the 100 most recent
pings for every check. You can set the limit higher to keep a longer history:
go to the Administration Panel, look up user's **Profile** and modify its
"Ping log limit" field.

Healthchecks provides management commands for cleaning up
`auth_user` (user accounts) and `api_tokenbucket` (rate limiting records) tables,
and for removing stale objects from external object storage.

Remove user accounts that are older than 1 month and have never logged in:

```sh
$ ./manage.py pruneusers
```

Remove old records from the `api_tokenbucket` table. The TokenBucket
model is used for rate-limiting login attempts and similar operations.
Any records older than one day can be safely removed.

```sh
$ ./manage.py prunetokenbucket
```

Remove old objects from external object storage. When an user removes
a check, removes a project, or closes their account, Healthchecks
does not remove the associated objects from the external object
storage on the fly. Instead, you should run `pruneobjects` occasionally
(for example, once a month). This command first takes an inventory
of all checks in the database, and then iterates over top-level
keys in the object storage bucket, and deletes any that don't also
exist in the database.

```sh
$ ./manage.py pruneobjects
```

When you first try these commands on your data, it is a good idea to
test them on a copy of your database, not on the live database right away.
In a production setup, you will want to run these commands regularly, as well as
have regular, automatic database backups set up.

## Next Steps

Get the [source code](https://github.com/healthchecks/healthchecks).

See [Configuration](../self_hosted_configuration/) for a list of configuration options.

