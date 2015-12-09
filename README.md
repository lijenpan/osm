# Manually building a tile server (14.04)
This page describes how to install, setup and configure all the necessary software to operate your own tile server. The step-by-step instructions are written for Ubuntu Linux 14.04 LTS (Trusty Tahr).
## Software Installation
The OSM tile server stack is a collection of programs and libraries that work together to create a tile server. As so often with OpenStreetMap, there are many ways to achieve this goal and nearly all of the components have alternatives that have various specific advantages and disadvantages. This tutorial describes the most standard version that is also used on the main OpenStreetMap.org tile server.

It consists of 5 main components: Mod_tile, renderd, mapnik, osm2pgsql and a postgresql/postgis database. Mod_tile is an apache module, that serves cached tiles and decides which tiles need re-rendering – either because they are not yet cached or because they are outdated. Renderd provides a priority queueing system for rendering requests to manage and smooth out the load from rendering requests. Mapnik is the software library that does the actual rendering and is used by renderd.

In order to build these components, a variety of dependencies need to be installed first:

<pre><code>sudo apt-get install libboost-all-dev subversion git-core tar unzip wget bzip2 build-essential autoconf libtool libxml2-dev libgeos-dev libgeos++-dev libpq-dev libbz2-dev libproj-dev munin-node munin libprotobuf-c0-dev protobuf-c-compiler libfreetype6-dev libpng12-dev libtiff4-dev libicu-dev libgdal-dev libcairo-dev libcairomm-1.0-dev apache2 apache2-dev libagg-dev ttf-unifont libgeotiff-epsg node-carto mapnik-utils</code></pre>

### Installing postgresql / postgis

On Ubuntu there are pre-packaged versions of both postgis and postgresql, so these can simply be installed via the Ubuntu package manager.
<pre><code>sudo apt-get install postgresql postgresql-contrib postgis postgresql-9.3-postgis-2.1</code></pre>

Now you need to create a postgis database. The defaults of various programs assume the database is called gis and we will use the same convention in this tutorial, although this is not necessary. Substitute your username for username in the two places below. This should be the username that will render maps with Mapnik.
<pre><code>sudo -u postgres -i
createuser [username] # answer yes for superuser (although this isn't strictly necessary)
createdb -E UTF8 -O [username] gis
exit
</code></pre>

### Create a Unix user for this user, too, choosing a password when prompted
<pre><code>sudo useradd -m [username]
sudo passwd [username]
</code></pre>

### Set up PostGIS on the PostgreSQL database
<pre><code>sudo -u postgres psql
\c gis
CREATE EXTENSION postgis;
CREATE EXTENSION postgis_topology;
ALTER TABLE geometry_columns OWNER TO [username];
ALTER TABLE spatial_ref_sys OWNER TO [username];
\q
</code></pre>

### Install Mapnik library
Next, we need to install the Mapnik library. Mapnik is used to render the OpenStreetMap data into the tiles used for an OpenLayers web map.

#### Optional steps if you are installing Mapnik 3.0.9
Default Ubuntu distro does not come with latest harfbuzz or boost libraries, which are required for Mapnik 3.0.9.

##### Install Haffbuzz 1.1.2
<pre><code>wget http://www.freedesktop.org/software/harfbuzz/release/harfbuzz-1.1.2.tar.bz2
tar xf harfbuzz-1.1.2.tar.bz2
cd harfbuzz-1.1.2
./configure && make && sudo make install
sudo ldconfig
</code></pre>

##### Install boost 1.59
<pre><code>http://downloads.sourceforge.net/boost/boost_1_59_0.tar.bz2
tar xf boost_1_59_0.tar.bz2
cd boost_1_59_0
sed -e '1 i#ifndef Q_MOC_RUN' \
    -e '$ a#endif'            \
    -i boost/type_traits/detail/has_binary_operator.hpp &&
./bootstrap.sh --prefix=/usr &&
./b2 stage threading=multi link=shared
</code></pre>

#### Build the Mapnik library from source
You would want to download 2.3.x branch.
<pre><code>git clone git://github.com/mapnik/mapnik
cd mapnik
./configure
make && sudo make install
</code></pre>

### Installing osm2pgsql
First install the dependencies:
<pre><code>sudo apt-get install make cmake g++ libboost-dev libboost-system-dev libboost-filesystem-dev libexpat1-dev zlib1g-dev libbz2-dev libpq-dev libgeos-dev libgeos++-dev libproj-dev lua5.2 liblua5.2-dev</code></pre>

osm2pgsql is under active development and is best compiled from source:
<pre><code>git clone git://github.com/openstreetmap/osm2pgsql.git
cd osm2pgsql
# If you are compiling master branch
mkdir build && cd build && cmake ..
make && sudo make install

# If you are compiling 0.88.x branch
./autogen.sh
./configure && make && sudo make install
</code></pre>

Download shapefiles manually:
* http://data.openstreetmapdata.com/simplified-land-polygons-complete-3857.zip
* http://data.openstreetmapdata.com/land-polygons-split-3857.zip
* http://planet.openstreetmap.org/historical-shapefiles/world_boundaries-spherical.tgz
* http://www.naturalearthdata.com/http//www.naturalearthdata.com/download/110m/cultural/ne_110m_admin_0_boundary_lines_land.zip
* http://data.openstreetmapdata.com/antarctica-icesheet-polygons-3857.zip
* http://data.openstreetmapdata.com/antarctica-icesheet-outlines-3857.zip

Put these shapefiles at <code>/usr/local/share/maps/style/data</code>.

To improve performance, we create index files for the larger shapefiles:
<pre><code>shapeindex --shape_files \
/usr/local/share/maps/style/data/land-polygons-split-3857/land_polygons.shp \
/usr/local/share/maps/style/data/simplified-land-polygons-complete-3857/simplified_land_polygons.shp \
/usr/local/share/maps/style/data/antarctica-icesheet-polygons-3857/icesheet_polygons.shp \
/usr/local/share/maps/style/data/antarctica-icesheet-outlines-3857/icesheet_outlines.shp \
/usr/local/share/maps/style/data/ne_110m_admin_0_boundary_lines_land/ne_110m_admin_0_boundary_lines_land.shp</code></pre>

Import your desired PBF of OSM data: <code>sudo -u [username] osm2pgsql --slim -d gis path/to/data.osm.pbf --style path/to/openstreetmap-carto.style</code>

Compile stylesheet for renderd: <code>carto path/to/openstreetmap-carto/project.mml > osm.xml</code>

Move osm.xml to <code>/usr/local/share/maps/style/</code>

### Install mod_tile and renderd
Compile the mod_tile source code:
<pre><code>git clone git://github.com/openstreetmap/mod_tile.git
cd mod_tile
./autogen.sh
./configure
make
sudo make install
sudo make install-mod_tile
sudo ldconfig
</code></pre>

### Setting up your webserver
Next we need to plug renderd and mod_tile into the Apache webserver, ready to receive tile requests.

#### Configure renderd
Change the the renderd settings by editing the /usr/local/etc/renderd.conf and change the following five lines, uncommenting (removing the ‘;’) when required. They are found in the [renderd], [mapnik] and [default] sections.

<pre><code>socketname=/var/run/renderd/renderd.sock
plugins_dir=/usr/local/lib/mapnik/input
font_dir=/usr/share/fonts/truetype/ttf-dejavu
XML=/usr/local/share/maps/style/osm.xml
HOST=localhost</code></pre>

Delete the rest of commented out lines.

Create the files required for the mod_tile system to run (remember to change username to your user’s name):
<pre><code>sudo mkdir /var/run/renderd
sudo chown [username] /var/run/renderd
sudo mkdir /var/lib/mod_tile
sudo chown [username] /var/lib/mod_tile</code></pre>

#### Configure mod_tile
Next, we need to tell the Apache web server about our new mod_tile installation.
Using your favourite text editor, create the file /etc/apache2/conf-available/mod_tile.conf and add one line:
<pre><code># This is the Apache server configuration file for providing OSM tile support
# through mod_tile

LoadModule tile_module /usr/lib/apache2/modules/mod_tile.so

<VirtualHost *:80>
    ServerName tile.openstreetmap.org
    ServerAlias a.tile.openstreetmap.org b.tile.openstreetmap.org c.tile.openstreetmap.org d.tile.openstreetmap.org
    DocumentRoot /var/www/html

# Specify the default base storage path for where tiles live. A number of different storage backends
# are available, that can be used for storing tiles.  Currently these are a file based storage, a memcached
# based storage and a RADOS based storage.
# The file based storage uses a simple file path as its storage path ( /path/to/tiledir )
# The RADOS based storage takes a location to the rados config file and a pool name ( rados://poolname/path/to/ceph.conf )
# The memcached based storage currently has no configuration options and always connects to memcached on localhost ( memcached:// )
#
# The storage path can be overwritten on a style by style basis from the style TileConfigFile
    ModTileTileDir /var/lib/mod_tile

# You can either manually configure each tile set with the default png extension and mimetype
#    AddTileConfig /folder/ TileSetName

# or manually configure each tile set, specifying the file extension 
#    AddTileMimeConfig /folder/ TileSetName js

# or load all the tile sets defined in the configuration file into this virtual host.
# Some tile set specific configuration parameters can only be specified via the configuration file option
    LoadTileConfigFile /usr/local/etc/renderd.conf

# Specify if mod_tile should keep tile delivery stats, which can be accessed from the URL /mod_tile
# The default is On. As keeping stats needs to take a lock, this might have some performance impact,
# but for nearly all intents and purposes this should be negligable ans so it is safe to keep this turned on.
    ModTileEnableStats On

# Turns on bulk mode. In bulk mode, mod_tile does not request any dirty tiles to be rerendered. Missing tiles
# are always requested in the lowest priority. The default is Off.
    ModTileBulkMode Off

# Timeout before giving up for a tile to be rendered
    ModTileRequestTimeout 3

# Timeout before giving up for a tile to be rendered that is otherwise missing
    ModTileMissingRequestTimeout 10

# If tile is out of date, don't re-render it if past this load threshold (users gets old tile)
    ModTileMaxLoadOld 16

# If tile is missing, don't render it if past this load threshold (user gets 404 error)
    ModTileMaxLoadMissing 50

# Sets how old an expired tile has to be to be considered very old and therefore get elevated priority in rendering
    ModTileVeryOldThreshold 31536000000000

# Unix domain socket where we connect to the rendering daemon
    ModTileRenderdSocketName /var/run/renderd/renderd.sock

# Alternatively you can use a TCP socket to connect to renderd. The first part
# is the location of the renderd server and the second is the port to connect to.
#   ModTileRenderdSocketAddr renderd.mydomain.com 7653

##
## Options controlling the cache proxy expiry headers. All values are in seconds.
##
## Caching is both important to reduce the load and bandwidth of the server, as
## well as reduce the load time for the user. The site loads fastest if tiles can be
## taken from the users browser cache and no round trip through the internet is needed.
## With minutely or hourly updates, however there is a trade-off between cacheability
## and freshness. As one can't predict the future, these are only heuristics, that
## need tuning.
## If there is a known update schedule such as only using weekly planet dumps to update the db,
## this can also be taken into account through the constant PLANET_INTERVAL in render_config.h
## but requires a recompile of mod_tile

## The values in this sample configuration are not the same as the defaults
## that apply if the config settings are left out. The defaults are more conservative
## and disable most of the heuristics.


##
## Caching is always a trade-off between being up to date and reducing server load or
## client side latency and bandwidth requirements. Under some conditions, like poor
## network conditions it might be more important to have good caching rather than the latest tiles.
## Therefor the following config options allow to set a special hostheader for which the caching
## behaviour is different to the normal heuristics
##
## The CacheExtended parameters overwrite all other caching parameters (including CacheDurationMax)
## for tiles being requested via the hostname CacheExtendedHostname
#ModTileCacheExtendedHostname cache.tile.openstreetmap.org
#ModTileCacheExtendedDuration 2592000

# Upper bound on the length a tile will be set cacheable, which takes
# precedence over other settings of cacheing
ModTileCacheDurationMax 604800

# Sets the time tiles can be cached for that are known to by outdated and have been
# sent to renderd to be rerendered. This should be set to a value corresponding
# roughly to how long it will take renderd to get through its queue. There is an additional
# fuzz factor on top of this to not have all tiles expire at the same time
ModTileCacheDurationDirty 900

# Specify the minimum time mod_tile will set the cache expiry to for fresh tiles. There
# is an additional fuzz factor of between 0 and 3 hours on top of this.
ModTileCacheDurationMinimum 10800

# Lower zoom levels are less likely to change noticeable, so these could be cached for longer
# without users noticing much.
# The heuristic offers three levels of zoom, Low, Medium and High, for which different minimum
# cacheing times can be specified.

#Specify the zoom level below  which Medium starts and the time in seconds for which they can be cached
ModTileCacheDurationMediumZoom 13 86400

#Specify the zoom level below which Low starts and the time in seconds for which they can be cached
ModTileCacheDurationLowZoom 9 518400

# A further heuristic to determine cacheing times is when was the last time a tile has changed.
# If it hasn't changed for a while, it is less likely to change in the immediate future, so the
# tiles can be cached for longer.
# For example, if the factor is 0.20 and the tile hasn't changed in the last 5 days, it can be cached
# for up to one day without having to re-validate.
ModTileCacheLastModifiedFactor 0.20

## Tile Throttling
## Tile scrappers can often download large numbers of tiles and overly straining tileserver resources
## mod_tile therefore offers the ability to automatically throttle requests from ip addresses that have
## requested a lot of tiles.
## The mechanism uses a token bucket approach to shape traffic. I.e. there is an initial pool of n tiles
## per ip that can be requested arbitrarily fast. After that this pool gets filled up at a constant rate
## The algorithm has two metrics. One based on overall tiles served to an ip address and a second one based on
## the number of requests to renderd / tirex to render a new tile. 

## Overall enable or disable tile throttling
ModTileEnableTileThrottling Off
# Specify if you want to use the connecting IP for throtteling, or use the X-Forwarded-For header to determin the
# IP address to be used for tile throttling. This can be useful if you have a reverse proxy / http accellerator
# in front of your tile server.
# 0 - don't use X-Forward-For and allways use the IP that apache sees
# 1 - use the client IP address, i.e. the first entry in the X-Forwarded-For list. This works through a cascade of proxies.
#     However, as the X-Forwarded-For is written by the client this is open to manipulation and can be used to circumvent the throttling
# 2 - use the last specified IP in the X-Forwarded-For list. If you know all requests come through a reverse proxy
#     that adds an X-Forwarded-For header, you can trust this IP to be the IP the reverse proxy saw for the request
ModTileEnableTileThrottlingXForward 0
## Parameters (poolsize in tiles and topup rate in tiles per second) for throttling tile serving. 
ModTileThrottlingTiles 10000 1 
## Parameters (poolsize in tiles and topup rate in tiles per second) for throttling render requests. 
ModTileThrottlingRenders 128 0.2

###
###    
# increase the log level for more detailed information
    LogLevel debug
</VirtualHost></code></pre>

Tell Apache that you have added the new module, and restart it:
<pre><code>sudo a2enconf mod_tile
service apache2 reload</code></pre>

### Tuning your system
A tile server can put a lot of load on hard- and software. The default settings may therefore not be appropriate and a significant improvement can potentially be achieved through tuning various parameters.

#### Tuning postgresql
The default configuration for PostgreSQL 9.3 needs to be tuned for the amount of data you are about to add to it. Edit the file /etc/postgresql/9.3/main/postgresql.conf and make the following changes:
<pre><code>shared_buffers = 128MB
checkpoint_segments = 20
maintenance_work_mem = 256MB
autovacuum = off</code></pre>

These changes require a kernel configuration change, which needs to be applied every time that the computer is rebooted. As root, edit /etc/sysctl.conf and add these lines near the top after the other “kernel” definitions:
<pre><code># Increase kernel shared memory segments - needed for large databases
kernel.shmmax=268435456</code></pre>

Reboot your computer. Run this:
<pre><code>sudo sysctl kernel.shmmax</code></pre>

and verify that it displays as 268435456.

### Loading data into your server
#### Get the latest OpenStreetMap data
Retrieve a piece of OpenStreetMap data in PBF format from http://planet.openstreetmap.org/.
If you need the entire planet file, you can do it by issuing the following command:
<pre><code>mkdir /usr/local/share/maps/planet
cd /usr/local/share/maps/planet
wget http://planet.openstreetmap.org/pbf/planet-latest.osm.pbf</code></pre>

Since the whole planet is at least 18GB when compressed, there are links to smaller country or state sized extracts on that page. However, many people will only need one country or city; you can download PBF files for these (‘extracts’) from download.geofabrik.de. We would recommend that you test with smaller areas, and only move up to the full planet when you are confident your setup is working.

#### Importing data into the database
With the conversion tool compiled and the database prepared, the following command will insert the OpenStreetMap data you downloaded earlier into the database. This step is very disk I/O intensive; the full planet will take anywhere from 10 hours on a fast server with SSDs to several days depending on the speed of the computer performing the import. For smaller extracts the import time is much faster accordingly, and you may need to experiment with different -C values to fit within your machine’s available memory.

You will need to run this command as a known Postgres user.
<pre><code>osm2pgsql --slim -d gis -C 1600 --number-process 3 -S /usr/local/share/osm2pgsql/default.style planet -latest.osm.pbf</code></pre>

You will see status report as it imports map tiles.
<pre><code>osm2pgsql SVN version 0.89.0-dev (64 bit id space)

Using built-in tag processing pipeline
Using projection SRS 900913 (Spherical Mercator)
Setting up table: planet_osm_point
Setting up table: planet_osm_line
Setting up table: planet_osm_polygon
Setting up table: planet_osm_roads
Allocating memory for dense node cache
Allocating dense node cache in one big chunk
Allocating memory for sparse node cache
Sharing dense sparse
Node-cache: cache=1600MB, maxblocks=25600*65536, allocation method=11
Mid: pgsql, scale=100 cache=1600
Setting up table: planet_osm_nodes
Setting up table: planet_osm_ways
Setting up table: planet_osm_rels

Reading in file: planet-latest.osm.pbf
Using PBF parser.
Processing: Node(37210k 383.6k/s) Way(0k 0.00k/s) Relation(0 0.00/s)</code></pre>

### Testing your tileserver
Now that everything is installed, set-up and loaded, you can start up your tile server and hopefully everything is working. We’ll run it interactively first, just to make sure that everything’s working properly. Remember to substitute your username again:
<pre><code>sudo -u [username] renderd -f -c /usr/local/etc/renderd.conf</code></pre>

and on a different session:
<code>service apache2 reload</code>

If any FATAL errors occur you’ll need to double-check any edits that you made earlier.
If not, try and browse to http://yourserveraddress/osm_tiles/0/0/0.png to see if a small picture of the world appears. The actual map tiles are being created as “metatiles” beneath the folder /var/lib/mod_tile.

### Setting it to run automatically
If it ran successfully, you can stop the interactive renderd process and configure it to run automatically at machine startup as a daemon.
<pre><code>sudo cp  ~/src/mod_tile/debian/renderd.init /etc/init.d/renderd
sudo chmod u+x /etc/init.d/renderd</code></pre>

Edit the /etc/init.d/renderd file as root – you’ll need to make a couple of changes to the DAEMON and DAEMON_ARGS lines so that they read:
<pre><code>DAEMON=/usr/local/bin/$NAME
DAEMON_ARGS="-c /usr/local/etc/renderd.conf"</code></pre>

Also, you’ll need to change references to www-data so that they match your username – change “www-data” to what you changed “username” to in other files.

You should now be able to start mapnik by doing the following:
<code>sudo /etc/init.d/renderd start</code>

and stop it:
<code>sudo /etc/init.d/renderd stop</code>

Logging information is now written to /var/log/syslog instead of to the terminal.

Next, add a link to the interactive startup directory so that it starts automatically:
<code>sudo ln -s /etc/init.d/renderd /etc/rc2.d/S20renderd</code>

and then restart your server, browse to http://yourserveraddress/osm_tiles/0/0/0.png and everything should be working! You can also go to the page http://yourserveraddress/mod_tile which should give you some stats about your tile server.
