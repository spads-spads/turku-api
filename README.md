# turku-api

## About Turku
Turku is an agent-based backup system where the server doing the backups has no direct access to the machine being backed up.  Instead, the machine's agent coordinates with an API server and opens a reverse tunnel to the storage server when it is time to do a backup.

Turku is comprised of the following components:

* [turku-api](https://launchpad.net/turku/turku-api), a Django web application which acts as an API server and coordinator.
* [turku-storage](https://launchpad.net/turku/turku-storage), an agent installed on the servers which do the actual backups.
* [turku-agent](https://launchpad.net/turku/turku-agent), an agent installed on the machines to be backed up.

turku-api has the following models:

* Auth, registration secrets for Machines and Storages.
* Storage, registered agents for servers running turku-storage.
* Machine, registered agents for machines running turku-agent.
* Source, sources of data to be backed up on a Machine.
* FilterSet, rsync filter definitions of what to include and exclude (optional).
* BackupLog, records of Machine/Source backup runs.

## Installation

turku-api is a standard Django application; please see [Django's installation guide](https://docs.djangoproject.com/en/1.11/topics/install/) for setting up an application.  Django 1.6 (Ubuntu Trusty era) through Django 1.11 (Ubuntu Bionic era) have been tested.

turku-api requires Python 3.  The following optional Python modules can also be installed:

* croniter, for cron-style scheduling definitions

It is highly recommended to serve turku-api over HTTPS, as registration and agent-specific secrets are passed from turku-storage and turku-agent agents to turku-api.  However, actual backups are done over SSH, not this application.

turku-api's default configuration will use a SQLite database.  This is fine for small installations, but it's recommended to use a PostgreSQL/MySQL database for larger installations.

Besides database configuration, turku-api's default Django settings are adequate.  The only recommended change is an installation-specific Django SECRET_KEY.  Create ```turku_api/local_settings.py``` with:

```
SECRET_KEY = 'long random key string'
```

Any other changes to ```turku_api/settings.py``` should be put in ```turku_api/local_settings.py``` as well.

Getting the admin web site working is recommended, but not required.

## Configuration

Once the installation is complete, you will need to add at least one Storage registration Auth, and at least one Machine registration Auth:

```
# python3 manage.py shell

from django.contrib.auth import hashers
from turku_api.models import Auth

storage_secret = hashers.get_random_string(30)
storage_auth = Auth()
storage_auth.active = True
storage_auth.name = 'Storages'
storage_auth.secret_hash = hashers.make_password(storage_secret)
storage_auth.secret_type = 'storage_reg'
storage_auth.save()

machine_secret = hashers.get_random_string(30)
machine_auth = Auth()
machine_auth.active = True
machine_auth.name = 'Machines'
machine_auth.secret_hash = hashers.make_password(machine_secret)
machine_auth.secret_type = 'machine_reg'
machine_auth.save()

print('Storage registration secret is: {}'.format(storage_secret))
print('Machine registration secret is: {}'.format(machine_secret))
```

Multiple of each type of Auth can be created.  For example, you may have multiple groups within an organization; creating a separate Machine registration Auth for each group may be desired.  turku-api tracks which Auth is used to register a Machine, but the Auth is not used for authentication beyond registration; a Machine-specific generated secret is used for check-ins.

## Deployments

Once you have the Storage and Machine registration secrets, move on to installing turku-storage and turku-agent agents and registering them with turku-api.  See the README.md files in their respective repositories for more details.

For small deployments, turku-api, turku-storage, and a turku-agent agent may all be on the same server.  For example, the server may have a main turku-api coordinator, a turku-storage agent configured to store backups on a removable USB drive, and a turku-agent agent configured to back up the turku-api SQLite database.

For larger deployments, Turku supports scale-out.  Load-balancing HTTPS frontends may point to multiple turku-api application servers, which themselves may use primary/secondary PostgreSQL databases.  Multiple turku-storage servers may register with turku-api, as storage needs increase.  All turku-agent machines will need access to the turku-api application servers (or their frontends) via HTTPS, and to their assigned turku-storage server over SSH.  No ingress access is needed to turku-agent machines.