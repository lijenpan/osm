# Manually building a tile server (14.04)
This page describes how to install, setup and configure all the necessary software to operate your own tile server. The step-by-step instructions are written for Ubuntu Linux 14.04 LTS (Trusty Tahr).
## Software Installation
The OSM tile server stack is a collection of programs and libraries that work together to create a tile server. As so often with OpenStreetMap, there are many ways to achieve this goal and nearly all of the components have alternatives that have various specific advantages and disadvantages. This tutorial describes the most standard version that is also used on the main OpenStreetMap.org tile server.

It consists of 5 main components: Mod_tile, renderd, mapnik, osm2pgsql and a postgresql/postgis database. Mod_tile is an apache module, that serves cached tiles and decides which tiles need re-rendering – either because they are not yet cached or because they are outdated. Renderd provides a priority queueing system for rendering requests to manage and smooth out the load from rendering requests. Mapnik is the software library that does the actual rendering and is used by renderd.

In order to build these components, a variety of dependencies need to be installed first:

<pre><code>sudo apt-get install libboost-all-dev subversion git-core tar unzip wget bzip2 build-essential autoconf libtool libxml2-dev libgeos-dev libgeos++-dev libpq-dev libbz2-dev libproj-dev munin-node munin libprotobuf-c0-dev protobuf-c-compiler libfreetype6-dev libpng12-dev libtiff4-dev libicu-dev libgdal-dev libcairo-dev libcairomm-1.0-dev apache2 apache2-dev libagg-dev ttf-unifont libgeotiff-epsg node-carto libharfbuzz</code></pre>

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
ALTER TABLE geometry_columns OWNER TO [username];
ALTER TABLE spatial_ref_sys OWNER TO [username];
\q
</code></pre>

### Installing osm2pgsql
First install the dependencies:
<pre><code>sudo apt-get install make cmake g++ libboost-dev libboost-system-dev libboost-filesystem-dev libexpat1-dev zlib1g-dev libbz2-dev libpq-dev libgeos-dev libgeos++-dev libproj-dev lua5.2 liblua5.2-dev</code></pre>

osm2pgsql is under active development and is best compiled from source:
<pre><code>git clone git://github.com/openstreetmap/osm2pgsql.git
cd osm2pgsql
mkdir build && cd build && cmake ..
make
sudo make install</code></pre>

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
cd maplink
./configure
make && sudo make install
</code></pre>

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

### Stylesheet configuration
To begin with, we need to download both the OSM Bright stylesheet, and also the additional data resources it uses (for coastlines and the like).
<pre><code>mkdir -p /usr/local/share/maps/style
cd /usr/local/share/maps/style
wget https://github.com/mapbox/osm-bright/archive/master.zip
wget http://data.openstreetmapdata.com/simplified-land-polygons-complete-3857.zip
wget http://data.openstreetmapdata.com/land-polygons-split-3857.zip
wget http://www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/cultural/ne_10m_populated_places_simple.zip
</code></pre>

We then move the downloaded data into the osm-bright-master project directory:
<pre><code>unzip '*.zip'
mkdir osm-bright-master/shp
mv land-polygons-split-3857 osm-bright-master/shp/
mv simplified-land-polygons-complete-3857 osm-bright-master/shp/
mv ne_10m_populated_places_simple osm-bright-master/shp/
</code></pre>

To improve performance, we create index files for the larger shapefiles:
<pre><code>cd osm-bright-master/shp/land-polygons-split-3857
shapeindex land_polygons.shp
cd ../simplified-land-polygons-complete-3857/
shapeindex simplified_land_polygons.shp
cd ..</code></pre>

#### Configuring OSM Bright
The OSM Bright stylesheet now needs to be adjusted to include the location of our data files. Edit the file osm-bright/osm-bright.osm2pgsql.mml in your favourite text editor, for example:
<pre><code>vim osm-bright/osm-bright.osm2pgsql.mml</code></pre>

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

#### Compiling the stylesheet
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

### Setting up your webserver
Next we need to plug renderd and mod_tile into the Apache webserver, ready to receive tile requests.

#### Configure renderd
Change the the renderd settings by editing the /usr/local/etc/renderd.conf and change the following five lines, uncommenting (removing the ‘;’) when required. They are found in the [renderd], [mapnik] and [default] sections.

<pre><code>socketname=/var/run/renderd/renderd.sock
plugins_dir=/usr/local/lib/mapnik/input
font_dir=/usr/share/fonts/truetype/ttf-dejavu
XML=/usr/local/share/maps/style/OSMBright/OSMBright.xml
HOST=localhost</code></pre>

Create the files required for the mod_tile system to run (remember to change username to your user’s name):
<pre><code>sudo mkdir /var/run/renderd
sudo chown [username] /var/run/renderd
sudo mkdir /var/lib/mod_tile
sudo chown [username] /var/lib/mod_tile</code></pre>

#### Configure mod_tile
Next, we need to tell the Apache web server about our new mod_tile installation.
Using your favourite text editor, create the file /etc/apache2/conf-available/mod_tile.conf and add one line:
<pre><code>LoadModule tile_module /usr/lib/apache2/modules/mod_tile.so</code></pre>

Apache’s default website configuration file needs to be modified to include mod_tile settings. Modify the file /etc/apache2/sites-available/000-default.conf to include the following lines immediately after the admin e-mail address line:
<pre><code>LoadTileConfigFile /usr/local/etc/renderd.conf
ModTileRenderdSocketName /var/run/renderd/renderd.sock
# Timeout before giving up for a tile to be rendered
ModTileRequestTimeout 0
# Timeout before giving up for a tile to be rendered that is otherwise missing
ModTileMissingRequestTimeout 30</code></pre>

Tell Apache that you have added the new module, and restart it:
<pre><code>a2enconf mod_tile
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
<pre><code>sudo mkdir /var/run/renderd
sudo chown [username] /var/run/renderd
sudo -u [username] renderd -f -c /usr/local/etc/renderd.conf</code></pre>

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
