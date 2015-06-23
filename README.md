# Setting Up OpenStreetMap Tile Server with Official Styles on AWS EC2 with PostgreSQL RDS and PostGIS
These are some notes on setting up OpenStreetMap tile server with official OSM
styles on an Amazon EC2 instance with a PostgreSQL RDS instance supporting PostGIS.

These notes will cover some technical details of the AWS EC2 + RDS stack,
that are omitted or not present in the guides at https://switch2osm.org

_NOTE:_ This is written after the fact, I may have forgotten or omitted something
by accident.

## Overview

### AWS Setup Overview
* Amazon Linux AMI https://aws.amazon.com/amazon-linux-ami/2015.03-release-notes/
* m3.medium instance with 60G General Purpose SSD + 1 Elastic IP
* PostgreSQL RDS m3.medium instance with (default) 100G storage + PostGIS 2.1.5

### Setting Up the EC2 Instance
* http://aws.amazon.com/ec2/
* http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html

### Setting Up PostgreSQL RDS Instance with PostGIS
* http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html
* http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html

Before configuring the PostgreSQL instance, prepare your AMI Linux instance first
and refer to _Configure PostgresSQL RDS Instance_ further below.

## Prepare AMI Linux via Packages
To continue with this step your EC2 instance should be up and running and you should
be able to access it over SSH.

First, enable the EPEL repository for extra packages:
```
$ sudo yum-config-manager --enable epel 
```

To make life easier on the command line install [byobu](https://en.wikipedia.org/wiki/Byobu_\(software\))
and to be able to clone required OpenStreetMap software later, also install git
```
$ sudo yum install byobu git
```

Additional packages are required for building and configuring the OpenStreetMap
software stack. Some of the packages listed below will trigger additional dependencies
that are also to be installed.
```
$ sudo yum install \
  apr apr-devel \
  apr-util apr-util-devel \
  autoconf automake \
  boost boost-devel \
  bzip2 bzip2-devel \
  cairo cairo-devel \
  cjkuni-fonts-common \
  freetype-devel \
  gcc gcc-c++ \
  gdal gdal-devel \
  geos geos-devel \
  httpd24 httpd24-devel \
  libcurl libucrl-devel \
  libicu libicu-devel \
  libjpeg-turbo libjpeg-turbo-devel \
  libpng libpng-devel \
  libtiff libtiff-devel \
  libtool \
  libuv libuv-devel \
  libxml libxml-devel \
  lua lua-devel \
  mod24_ssl \
  nodejs npm \ 
  ogdi \
  postgresql93 postgresql93-contrib postgresql93-devel postgresql93-libs postgresql93-server \
  proj proj-devel \
  protobuf protobuf-c protobuf-c-devel protobuf-compiler protobuf-devel \
  python27 python27-PyYAML python27-pycairo python27-pycairo-devel \
  sqlite sqlite-devel \
  zlib zlib-devel
```

Phew, one last package, install [CartoCSS](https://github.com/mapbox/carto) which
we will require later to create map styles for Mapnik.
```
$ sudo npm install -g carto
```

In the home directory of your AMI Linux user, the ec2-user, create a src directory
```
$ mkdir src
```

The _~/src_ directory will be used going forward to get (git clone) additional software
required:
* OpenStreetMap Carto
* OpenStreetMap data into PostgreSQL converter (osm2pgsql)
* Mapnik
* mod_tile

## Configure PostgreSQL RDS Instance
The PostgreSQL instance should be set up with the user _postgres_ with a password
of your choosing and a default database named _gis_.

After logging in into your AMI Linux instance connect to PostgreSQL

```
$ psql -U postgres -H <your-server-zone>.rds.amazon.com gis
```

In the psql terminal follow the PostGIS set up instructions accordingly to your
database instance username.
* http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.PostgreSQL.CommonDBATasks.html

```
gis=> create extension postgis;
```

Should be the bare minimum (<- Office Space reference right here!)

In addition you could do

```
gis=> create extension fuzzystrmatch;
gis=> create extension postgis_tiger_geocoder;
gis=> create extension postgis_topology;
```

Give access and change ownership:

```
gis=> grant connection on database gis to postgres;
gis=> alter schema public owner to postgres;
```

and in addition to the bare minimum if you were feeling like it:

```
gis=> alter schema tiger owner to postgres;
gis=> alter schema tiger_data owner to postgres;
gis=> alter schema topology owner to postgres;
```

## OpenStreetMap Software Stack Installation

### Mapnik
* https://github.com/mapnik/mapnik

#### Clone & Build
```
$ cd ~/src
$ git clone git://github.com/mapnik/mapnik
$ cd mapnik
$ git checkout -b v2.2.0 v2.2.0
$ python scons/scons.py configure INPUT_PLUGINS=all OPTIMIZATION=3 SYSTEM_FONTS=/usr/share/fonts/dejavu/
$ make
$ sudo make install
```

_NOTE:_ the following optional dependencies could not be satisfied with default
AMI Linux packages.
```
Note: will build without these OPTIONAL dependencies:
   - ociei (Oracle database library | configure with OCCI_LIBS & OCCI_INCLUDES | more info: https://github.com/mapnik/mapnik/wiki/OCCI)
   - rasterlite
```

#### ldconfig
The mapnik libraries are installed into _/usr/local/lib_, which for some reason
is ignored by default by AMI Linux. A configuration file needs to be set up for
ldconfig to fix this.
```
$ pushd .
$ cd /etc/ld.so.conf.d/
$ sudo echo -n "/usr/local/lib" > mapnik.conf
$ sudo ldconfig
$ popd
```

#### Test Installation
```
$ python
Python 2.7.9 (default, Apr  1 2015, 18:18:03)
[GCC 4.8.2 20140120 (Red Hat 4.8.2-16)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import mapnik
>>> quit()
```
Between the _import mapnik_ and _quit()_ commands there should be no output,
indicating that the _libmapnik.so.2.2.0_ was properly found from _/usr/local/lib_


### OpenStreetMap Carto
These are the CartoCSS map style sheets for the Standard map layer on
https://www.openstreetmap.org/

* https://github.com/gravitystorm/openstreetmap-carto

```
$ cd ~/src
$ git clone https://github.com/gravitystorm/openstreetmap-carto
```

As of writing this the latest stable in the v2.x is v2.29.1

```
$ cd openstreetmap-carto
$ git checkout -b v2.29.1 v2.29.1
```

#### Configuring database credentials
With a text editor modify the _osm2pgsql:_ part in __parts_ section in the
project.yaml file by adding _host_, _user_ and _password_ key value pairs

```
    31   osm2pgsql: &osm2pgsql
    32     type: "postgis"
    33     dbname: "gis"
    34     host: "<your-server-zone>.rds.amazonaws.com"
    35     user: "postgres"
    36     password: "<your-postgres-password>"
    37     key_field: ""
    38     geometry_field: "way"
    39     extent: "-20037508,-20037508,20037508,20037508"
```

Now as described in _Editing Layers_ at https://github.com/gravitystorm/openstreetmap-carto/blob/master/CONTRIBUTING.md
populate the new database configuration to (a new) project.mml

```
$ ./scripts/yaml2mml.py < project.yaml > project-1.mml
```

#### Get Shapefiles
```
$ ./get-shapefiles.sh
```

After which the _data/_ sub-directory is created into the openstreetmap-carto directory.

#### Create Mapnik Configuration
```
$ carto project-1.mml > OSMDefault.xml
```
That is it for now, we will follow up on this a bit later, after having set up
the data set via osm2pgsql and built Mapnik and mod\_tile.


### OpenStreetMap data into PostgreSQL converter (osm2pgsql)
* https://github.com/openstreetmap/osm2pgsql

#### Clone & Build
```
$ cd ~/src
$ git clone git://github.com/openstreetmap/osm2pgsql.git osm2pgsql
$ cd osm2pgsql
$ ./autogen.sh
$ ./configure
$ make
$ sudo make install
```

As of writing this the osm2psql version is _0.87.4-dev_.

#### Get Your Data Set
Importing the whole planet data will take _forever_ on this m3.medium instance
used here, if that is your requirement you should upgrade your instance.

One of the sites offering OSM data sets is at http://download.geofabrik.de/

For demonstatration purposes we will use a single country from http://download.geofabrik.de/europe.html
You can pick yours :) Depending on the file size this can take a while.
```
$ cd ~/src
$ mkdir osm-data; cd osm-data
$ wget http://download.geofabrik.de/europe/estonia-latest.osm.pbf
```

#### Improting Data Into Postgres
As the m3.medium is a single CPU instance with only 3.7GB of RAM the import will
utilize the slim mode of osm2pgsql. For additional osm2pgsql help and options do:
```
$ osm2pgsql -h -v | less
```

To import the data set in slim mode with a cache of 2500MB, forcing a Postgres
password prompt and utlizing the style from OpenStreetMap Carto:
```
$ osm2pgsql -c -s -C 2500 -G -K -W -d gis -U postgres -H <your-server-zone>.rds.amazonaws.com norway-latest.osm.pbf -S ~/src/openstreetmap-carto/openstreetmap-carto.style
```

After a successful import the the table listing in psql should look something like this:
```
gis=> \dt
                List of relations
  Schema  |        Name        | Type  |  Owner   
----------+--------------------+-------+----------
 public   | planet_osm_line    | table | postgres
 public   | planet_osm_nodes   | table | postgres
 public   | planet_osm_point   | table | postgres
 public   | planet_osm_polygon | table | postgres
 public   | planet_osm_rels    | table | postgres
 public   | planet_osm_roads   | table | postgres
 public   | planet_osm_ways    | table | postgres
 public   | spatial_ref_sys    | table | rdsadmin
 topology | layer              | table | rdsadmin
 topology | topology           | table | rdsadmin
(10 rows)
```

### mod_tile
A program to efficiently render and serve map tiles for https://www.openstreetmap.org
map using Apache and Mapnik.

* https://github.com/openstreetmap/mod_tile

The mod\_tile will provide the following utilities that will be installed into
_/usr/local/bin_
* renderd - Mapnik rendering daemon
* render_list - for pre-rendering the map tiles

#### Clone & Build
```
$ cd ~/src
$ git clone git://github.com/openstreetmap/mod_tile.git
$ cd mod_tile
$ ./autogen.sh
$ ./configure
$ make
$ sudo make install
$ sudo make install-mod_tile
$ sudo ldconfig
```

#### Set Up Style Configuration
This will set up the OpenStreetMap Carto files to be used for Mapnik and renderd.
```
$ cd ~/src/openstreetmap-carto
$ sudo cp OSMDefault.xml /usr/local/share/maps/style/
$ sudo cp -R data/ /usr/local/share/maps/style/
$ sudo cp -R symbols/ /usr/local/share/maps/style/
```

#### Configure renderd
Locate the configuration file at _/usr/local/etc/renderd.conf_ and fire up your
text editor (sudo is required).

Locate the listed key/value pairs and un-comment them by removing the leading
; (semi-colon) and altering the values accordingly:
* socketname=/var/run/renderd/renderd.sock
* plugins_dir=/usr/local/lib/mapnik/input
* font_dir=/usr/share/fonts/dejavu
* XML=/usr/local/share/maps/style/OSMDefault.xml
* HOST=\<your-ec2-dashed-ip-zone\>.compute.amazonaws.com

So that the end result (with your editor's line numbers) might look something like this:
```
    1 [renderd]
    2 socketname=/var/run/renderd/renderd.sock
    .
    .
    .
   21 [mapnik]
   22 plugins_dir=/usr/local/lib/mapnik/input
   23 font_dir=/usr/share/fonts/dejavu
   24 font_dir_recurse=1
   25
   26 [default]
   27 URI=/osm_tiles/
   28 TILEDIR=/var/lib/mod_tile
   29 XML=/usr/local/share/maps/style/OSMDefault.xml
   30 HOST=<your-ec2-dashed-ip-zone>.compute.amazonaws.com
   31 TILESIZE=256
   .
   .
   .
```

Set up the directories required for the mod_tile system to run:
```
$ sudo mkdir /var/run/renderd
$ sudo chown ec2-user:apache /var/run/renderd
$ sudo chmod 0775 /var/run/renderd
$ sudo mkdir /var/lib/mod_tile
$ sudo chown ec2-user:apache /var/lib/mod_tile
$ sudo chmod 0755 /var/lib/mod_tile
```

### Apache HTTPD
The Apache HTTPD on AMI Linux is configured via several configuration files at
_/etc/httpd/_

_NOTE:_ Do not forget to configure port 80 (and 443 for TLS) for your EC2 instance
to make the httpd server available for the world.

#### mod_tile.so
The mod_tile.so file has already been deployed in _/etc/httpd/modules_, to
configure mod_tile for httpd:
```
$ cd /etc/httpd/conf.modules.d/
$ sudo echo - n "LoadModule tile_module modules/mod_tile.so" > 00-mod_tile.conf
```

#### httpd.conf
The renderd configuration can be added into httpd's main configuration file at
_/etc/httpd/conf/httpd.conf_.

* open the _httpd.conf_ with sudo in your text editor and locate the _ServerName_
  configuration option
* change _ServerName_ to match \<your-ec2-dashed-ip-zone>.compute.amazonaws.com
* insert the renderd configuration below
  
```
# renderd
LoadTileConfigFile /usr/local/etc/renderd.conf
ModTileRenderdSocketName /var/run/renderd/renderd.sock
# Timeout before giving up for a tile to be rendered
ModTileRequestTimeout 0
# Timeout before giving up for a tile to be rendered that is otherwise missing
ModTileMissingRequestTimeout 30
```

#### Test Configuration and Start
To run httpd:
```
$ sudo service httpd start
```

The command line outuput should look something like:
```
Starting httpd: [Thu Jun 18 06:03:12.519392 2015] [tile:notice] [pid 23221] Loading tile config default at /osm_tiles/ for zooms 0 - 20 from tile directory /var/lib/mod_tile with extension .png and mime type image/png
                                                           [  OK  ]
```

## The Tile Smoke Test
First start renderd (foreground)
```
$ sudo -u ec2-user renderd -f -c /usr/local/etc/renderd.conf
```

Open a browser in your computer and navigate to:
```
http://<your-ec2-dashed-ip-zone>.compute.amazonaws.com/osm_tiles/0/0/0.png
```
If all goes well you should see a small tile of Earth. Also, look at the renderd
output for troubleshooting issues.
