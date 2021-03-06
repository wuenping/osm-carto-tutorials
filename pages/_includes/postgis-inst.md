## Install PostgreSQL and PostGIS

[PostgreSQL](https://www.postgresql.org/) is a relational database, and [PostGIS](http://postgis.net/) is its spatial extender, which allow you to store geographic objects like map data in it. PostgreSQL + PostGIS are used for a wide variety of uses such as rendering maps, geocoding, and analysis. It serves a similar function to ESRI’s SDE or Oracle’s Spatial extension.

Currently the tested versions are PostgreSQL 9.5 and PostGIS 2.2:

Also older PostgreSQL version should be suitable.

On Ubuntu there are pre-packaged versions of both postgis and postgresql, so these can simply be installed via the Ubuntu package manager.

```shell
sudo apt-get update
sudo apt-get install -y postgresql postgis pgadmin3 postgresql-contrib
```

Note: used PostgeSQL port is 5432 (default).

A user named {{ pg_user }} will be created during the installation process.

## Set the password for the *{{ pg_user }}* user

```shell
sudo -u {{ pg_user }} psql postgres
\password {{ pg_user }}
```

Alternative procedure (useful if you get authentication issues with the previous one):

```shell
sudo su -
sudo -i -u {{ pg_user}}
psql {{ pg_user}}
\password {{ pg_user }}
```

Enter the following password twice: `{{ pg_password }}`

This is just an example of password, you can use the one you prefer.

After entering the password, exit from *psql* with:

    \q

With the second procedure, also isssue:

```shell
exit # from 'sudo -i -u {{ pg_user}}'
exit # from 'sudo su -'
```

## Create the PostGIS instance

Now you need to create a postgis database. The defaults of various programs including openstreetmap-carto (ref. project.mml) assume the database is called *gis*. You need to set up PostGIS on the PostgreSQL database.

```shell
export PGPASSWORD={{ pg_password }}
HOSTNAME=localhost # set it to the actual ip address or host name
psql -U {{ pg_user }} -h $HOSTNAME -c "create database gis" # alternative command: createdb -E UTF8 -O {{ pg_user }} gis
psql -U {{ pg_user }} -h $HOSTNAME -c "\connect gis"
psql -U {{ pg_user }} -h $HOSTNAME -d gis -c "CREATE EXTENSION postgis"
psql -U {{ pg_user }} -h $HOSTNAME -d gis -c "CREATE EXTENSION hstore"
```

The character encoding scheme to be used in the database is UTF8.

If you get the following error

    ERROR:  could not open extension control file "/usr/share/postgresql/9.3/extension/postgis.control": No such file or directory

then you might be installing PostgreSQL 9.3, for which you should also need:

    sudo apt-get install postgis postgresql-9.3-postgis-scripts

Install it and repeat the create extension commands.

## Add a user and grant access to gis DB

In order for the application to access the *gis* database, a DB user with the same name of your UNIX user is needed. Let's suppose your UNIX ue is *{{ pg_login }}*.

```shell
sudo su -
sudo -i -u {{ pg_user}}
createuser {{ pg_login }}
psql
grant all privileges on database gis to {{ pg_login }};
\q
exit
exit
```

## Enabling remote access to PostgreSQL

To remotely access PostgreSQL, you need to edit *pg_hba.conf*:

    sudo vi /etc/postgresql/9.5/main/pg_hba.conf

and add the following line:

    host    all             all             <your IP set>/<your netmask>             md5

`host all all 0.0.0.0/0 md5` is an access control rule that let anybody login in from any address if providing a valid password (md5 keyword).

Then edit *postgresql.conf*:

    sudo vi /etc/postgresql/9.5/main/postgresql.conf

and change the following line:

    set `listen_addresses = '*'`

Finally, the DB shall be restarted:

    sudo /etc/init.d/postgresql restart

## Tuning the database

The [PostgreSQL wiki](http://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server) has a page on database tuning. The important settings are shared_buffers, checkpoint_segments, and the write-ahead-log settings. There are also some settings you want to change specifically for the import.

The default PostgreSQL settings aren't great for very large databases like OSM databases. Proper tuning can just about double the performance you're getting. The most important PostgreSQL settings to change are `maintenance_work_mem` and `work_mem`, both which should be increased for faster data loading and faster queries while rendering respectively. Conservative settings for a 2GB VM are `work_mem=16MB` and `maintenance_work_mem=128MB`. On a machine with enough memory you could set them as high as `work_mem=128MB` and `maintenance_work_mem=1GB`. An overview to tuning PostgreSQL can be found on the [PostgreSQL Wiki](https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server), but adjusting `maintenance_work_mem` and `work_mem` are probably enough on a development or testing machine.[^98]

To edit the PostgreSQL configuration file with *vi* editor:

    sudo vi /etc/postgresql/9.5/main/postgresql.conf

and if you are running PostgreSQL 9.3:

    sudo vi /etc/postgresql/9.3/main/postgresql.conf

Suggested settings:

    shared_buffers = 128MB
    checkpoint_segments = 20
    maintenance_work_mem = 256MB
    autovacuum = off
    fsync = off

The latter two ones allow a faster import.

The first turns off auto-vacuum during the import and allows you to run a vacuum at the end. The second will introduce data corruption in case of a power outage and is dangerous. If you have a power outage while importing the data you will have to drop the data from the database and re-import, but it’s faster. Just remember to change these settings back after importing. fsync has no effect on query times once the data is loaded.

Suggested settings for a system with 16GB of RAM:

    shared_buffers = 2GB
    work_mem = 128MB
    maintenance_work_mem = 1GB
    wal_level = minimal
    synchronous_commit = off
    checkpoint_segments = 64
    checkpoint_timeout = 15min
    checkpoint_completion_target = 0.9
    default_statistics_target = 1000
    autovacuum = off
    fsync = off

To stop and start the database:

    sudo /etc/init.d/postgresql stop

    sudo /etc/init.d/postgresql start

You may get an error and need to increase the shared memory size. Edit */etc/sysctl.d/30-postgresql-shm.conf* and run `sudo sysctl -p /etc/sysctl.d/30-postgresql-shm.conf`. A parameter like `kernel.shmmax=17179869184` and `kernel.shmall=4194304` could be appropriate for a 16GB segment size.[^99]

## Install Osm2pgsql

Osm2pgsql is an OpenStreetMap specific software used to load the OSM data into the PostGIS database.

To install [osm2pgsql](https://wiki.openstreetmap.org/wiki/Osm2pgsql):

    sudo apt-get install -y osm2pgsql

Go to [Get an OpenStreetMap data extract](#get-an-openstreetmap-data-extract).

### Alternative installation procedure

This alternative installation procedure generates the most updated executable by compiling the sources.

```shell
# Needed dependencies
sudo apt-get install -y make cmake g++ libboost-dev libboost-system-dev \
  libboost-filesystem-dev libexpat1-dev zlib1g-dev \
  libbz2-dev libpq-dev libgeos-dev libgeos++-dev libproj-dev lua5.2 \
  liblua5.2-dev

# Download osm2pgsql
cd /tmp
git clone git://github.com/openstreetmap/osm2pgsql.git 

# Prepare for compiling
cd osm2pgsql
mkdir build && cd build
cmake ..

# Compile
make

# Install
sudo make install

# Clean-out temporary files
cd ../..
rm -rf osm2pgsql
```

{% include_relative _includes/download-osm-data.md %}

## Load data to PostGIS

```shell
cd {{ include.cdprogram }}
cd openstreetmap-carto
HOSTNAME=localhost # set it to the actual ip address or host name
osm2pgsql -s -C 300 -c -G -d gis --style openstreetmap-carto.style -H $HOSTNAME -U {{ pg_user }} [.osm or .pbf file]
```

Note: if you get the following error:

```shell
node_changed_mark failed: ERROR:  prepared statement "node_changed_mark" does not exist
```

do the following command on your *original.osm*:

```shell
sed "s/action='modify' //" < original.osm | > fixedfile.osm
```

Then process *fixedfile.osm*.

If you get the following error:

    Error reading style file line 79 (fields=4)
    flag 'phstore' is invalid in non-hstore mode
    Error occurred, cleaning up

then you need to add the *hstore* flag to *osm2pgsql*:

    osm2pgsql -s -C 300 -c -G -d gis --style openstreetmap-carto.style --hstore -H $HOSTNAME -U {{ pg_user }} [.osm or .pbf file]

## Create indexes

Add the indexes indicated by *openstreetmap-carto*:

```shell
HOSTNAME=localhost # set it to the actual ip address or host name
cd {{ include.cdprogram }}
cd openstreetmap-carto
scripts/indexes.py | psql -U {{ pg_user }} -h $HOSTNAME -d gis
```

Read [custom indexes](https://github.com/gravitystorm/openstreetmap-carto/blob/master/INSTALL.md#custom-indexes) for further information.

[^98]: Information taken from [switch2osm](https://switch2osm.org/loading-osm-data).
[^99]: [Information from Paul Norman's Blog](http://www.paulnorman.ca/blog/2011/11/loading-a-pgsnapshot-schema-with-a-planet-take-2/).
