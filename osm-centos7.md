# Preparations
<pre><code>sudo yum -y install bzip2-devel libpng-devel libtiff-devel zlib-devel libjpeg-devel libxml2-devel python-setuptools proj-devel proj proj-epsg proj-nad freetype-devel freetype libicu-devel libicu gdal-devel gdal sqlite-devel sqlite libcurl-devel libcurl cairo-devel cairo pycairo-devel pycairo geos geos-devel protobuf-devel protobuf-c-devel lua-devel cmake proj proj-devel</code></pre>

# Install PostgreSQL Server and PostGIS
<code>sudo yum install -y postgresql94-server postgis2_94 postgis2_94-docs postgis2_94-utils pgrouting_94</code>

## Initialize Postgresql DB
<pre><code>service postgresql-9.4 initdb
# Create DB: gis</code></pre>

## Start Postgresql Server
<pre><code>chkconfig postgresql-9.4 on
service postgresql-9.4 start</code></pre>

# Build and Install Mapnik
## Boost C++ Libraries
<pre><code>JOBS=`grep -c ^processor /proc/cpuinfo`
wget http://downloads.sourceforge.net/boost/boost_1_59_0.tar.bz2
tar xf boost_1_59_0.tar.bz2
cd boost_1_59_0
./bootstrap.sh
./b2 -d1 -j${JOBS} \
    --with-thread \
    --with-filesystem \
    --with-python \
    --with-regex -sHAVE_ICU=1  \
    --with-program_options \
    --with-system \
    link=shared \
    release \
    toolset=gcc \
    stage
./b2 -d1 -j${JOBS} \
    --with-thread \
    --with-filesystem \
    --with-python \
    --with-regex -sHAVE_ICU=1  \
    --with-program_options \
    --with-system \
    link=shared \
    release \
    toolset=gcc \
    install
    
# set up support for libraries installed in /usr/local/lib
sudo bash -c "echo '/usr/local/lib' > /etc/ld.so.conf.d/boost.conf"
sudo ldconfig</code></pre>

## Build Mapnik
### Build Preparation
Add pg_config to the PATH:
<code>export PATH=$PATH:/usr/pgsql-9.4/bin</code>

### Clone, Build, and Install Mapnik 2.3.x
<pre><code>git clone git://github.com/mapnik/mapnik
cd maplink
./configure
make && sudo make install</code></pre>

If you experience odd configuration errors, try cleaning the configure caches: <code>make clean</code>

# Install Geos C++ Library
<pre><code>wget http://download.osgeo.org/geos/geos-3.5.0.tar.bz2
tar xf geos-3.5.0.tar.bz2
cd geos-3.5.0
./configure && make && sudo make install
ldconfig
</code></pre>

Add the following line to /etc/ld.so.conf if it doesn't already exist: <code>/usr/local/lib</code>

# Install osm2pgsql
osm2pgsql is under active development and is best compiled from source:
<pre><code>git clone git://github.com/openstreetmap/osm2pgsql.git
cd osm2pgsql
mkdir build && cd build && cmake ..
make
sudo make install
export PATH=$PATH:/usr/local/bin</code></pre>

# Install mod_tile and renderd
Compile the mod_tile source code:
<pre><code>git clone git://github.com/openstreetmap/osm2pgsql.git
cd osm2pgsql
mkdir build && cd build && cmake ..
make
sudo make install
export PATH=$PATH:/usr/local/bin</code></pre>
git clone git://github.com/openstreetmap/mod_tile.git
cd mod_tile
./autogen.sh
./configure
make
sudo make install
sudo make install-mod_tile
sudo ldconfig</code></pre>

# Stylesheet configuration
To begin with, we need to download both the OSM Bright stylesheet, and also the additional data resources it uses (for coastlines and the like).
<pre><code>mkdir -p /usr/local/share/maps/style
cd /usr/local/share/maps/style
wget https://github.com/mapbox/osm-bright/archive/master.zip
wget http://data.openstreetmapdata.com/simplified-land-polygons-complete-3857.zip
wget http://data.openstreetmapdata.com/land-polygons-split-3857.zip
wget http://www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/cultural/ne_10m_populated_places_simple.zip</code></pre>

We then move the downloaded data into the osm-bright-master project directory:
<pre><code>unzip '*.zip'
mkdir osm-bright-master/shp
mv land-polygons-split-3857 osm-bright-master/shp/
mv simplified-land-polygons-complete-3857 osm-bright-master/shp/
mv ne_10m_populated_places_simple osm-bright-master/shp/</code></pre>

To improve performance, we create index files for the larger shapefiles:
<pre><code>cd osm-bright-master/shp/land-polygons-split-3857
shapeindex land_polygons.shp
cd ../simplified-land-polygons-complete-3857/
shapeindex simplified_land_polygons.shp
cd ..</code></pre>

## Configuring OSM Bright
The OSM Bright stylesheet now needs to be adjusted to include the location of our data files. Edit the file osm-bright/osm-bright.osm2pgsql.mml in your favourite text editor, for example:
<code>vim osm-bright/osm-bright.osm2pgsql.mml</code>

Find the lines with URLs pointing to shapefiles (ending .zip) and replace each one with these appropriate pairs of lines:
<pre><code>"file": "/usr/local/share/maps/style/osm-bright-master/shp/land-polygons-split-3857/land_polygons.shp", 
"type": "shape"</code></pre>
<pre><code>"file": "/usr/local/share/maps/style/osm-bright-master/shp/simplified-land-polygons-complete-3857/simplified_land_polygons.shp", 
"type": "shape",</code></pre>
<pre><code>"file": "/usr/local/share/maps/style/osm-bright-master/shp/ne_10m_populated_places_simple/ne_10m_populated_places_simple.shp", 
"type": "shape"</code></pre>

Note that we are also adding “type”: “shape” to each one.

Finally, in the section dealing with “ne_places”, replace the “srs” and “srs-name” lines with this one line:
<pre><code>"srs": "+proj=longlat +ellps=WGS84 +datum=WGS84 +no_defs"</code></pre>

## Compiling the stylesheet
We now have a fully working CartoCSS stylesheet. Before Mapnik can use it, we need to compile it into XML using the command-line carto compiler. First of all, we use OSM Bright’s own preprocessor, which we need to edit for our setup:
<pre><code>cp configure.py.sample configure.py
vim configure.py
</code></pre>

Change the config line pointing to ~/Documents/Mapbox/project to /usr/local/share/maps/style instead, and change dbname from osm to gis. Save and exit.

Run the pre-processor and then carto:
<pre><code>./make.py
cd ../OSMBright/
carto project.mml > OSMBright.xml
</code></pre>

You now have a Mapnik XML stylesheet at /usr/local/share/maps/style/OSMBright/OSMBright.xml.

# Setting up your webserver
Next we need to plug renderd and mod_tile into the Apache webserver, ready to receive tile requests.
## Configure renderd
Change the the renderd settings by editing the /etc/renderd.conf and change the following five lines, uncommenting (removing the ‘;’) when required. They are found in the [renderd], [mapnik] and [default] sections.
<pre><code>socketname=/var/run/renderd/renderd.sock
plugins_dir=/usr/local/lib/mapnik/input
font_dir=/usr/share/fonts/truetype/ttf-dejavu
XML=/usr/local/share/maps/style/OSMBright/OSMBright.xml
HOST=localhost</code></pre>

## Configure mod_tile
Next, we need to tell the Apache web server about our new mod_tile installation. Using your favourite text editor, create the file /etc/httpd/conf.d/mod_tile.conf and add one line:
<pre>LoadModule tile_module /etc/httpd/modules/mod_tile.so
&lt;VirtualHost *:80&gt;
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
    LoadTileConfigFile /etc/renderd.conf

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

&lt;/VirtualHost&gt;</pre>

Apache’s default website configuration file needs to be modified to include mod_tile settings. Modify the file /etc/apache2/sites-available/000-default.conf to include the following lines immediately after the admin e-mail address line:

# Tuning your system
A tile server can put a lot of load on hard- and software. The default settings may therefore not be appropriate and a significant improvement can potentially be achieved through tuning various parameters.

## Tuning postgresql
The default configuration for PostgreSQL 9.4 needs to be tuned for the amount of data you are about to add to it. Edit the file /var/lib/pgsql/9.4/data/postgresql.conf and make the following changes:
<pre><code>shared_buffers = 128MB
checkpoint_segments = 20
maintenance_work_mem = 256MB
autovacuum = off</code></pre>

These changes require a kernel configuration change, which needs to be applied every time that the computer is rebooted. As root, edit /etc/sysctl.conf and add these lines near the top after the other “kernel” definitions:
<pre><code># Increase kernel shared memory segments - needed for large databases
kernel.shmmax=268435456</code></pre>

Reboot your computer. Run this:
<code>sudo sysctl kernel.shmmax</code>
and verify that it displays as 268435456.

# Install OpenStreetMap
<pre><code>useradd -c "OpenStreetMap System User" -m osm
su - postgres
export PATH=$PATH:/usr/pgsql-9.4/bin
psql gis < /usr/pgsql-9.4/share/contrib/postgis-2.1/postgis.sql
psql gis < /usr/pgsql-9.4/share/contrib/postgis-2.1/spatial_ref_sys.sql
createuser osm -W # No to all questions
createuser apache -W # No to all questions
echo "grant all on geometry_columns to apache;" | psql gis
echo "grant all on spatial_ref_sys to apache;" | psql gis
exit
export PATH=$PATH:/usr/pgsql-9.4/bin
</code></pre>

## Get the latest OpenStreetMap data
Retrieve a piece of OpenStreetMap data in PBF format from http://planet.openstreetmap.org/. If you need the entire planet file, you can do it by issuing the following command:
<pre><code>mkdir /usr/local/share/maps/planet
cd /usr/local/share/maps/planet
wget http://planet.openstreetmap.org/pbf/planet-latest.osm.pbf</code></pre>

Since the whole planet is at least 18GB when compressed, there are links to smaller country or state sized extracts on that page. However, many people will only need one country or city; you can download PBF files for these (‘extracts’) from download.geofabrik.de. We would recommend that you test with smaller areas, and only move up to the full planet when you are confident your setup is working.

## Importing data into the database
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

## Testing your tileserver
Now that everything is installed, set-up and loaded, you can start up your tile server and hopefully everything is working. We’ll run it interactively first, just to make sure that everything’s working properly. Remember to substitute your username again:
<pre><code>./renderd -f -c /usr/local/etc/renderd.conf</code></pre>

and on a different session:
<code>service httpd reload</code>

If any FATAL errors occur you’ll need to double-check any edits that you made earlier.
If not, try and browse to http://yourserveraddress/osm_tiles/0/0/0.png to see if a small picture of the world appears. The actual map tiles are being created as “metatiles” beneath the folder /var/lib/mod_tile.
