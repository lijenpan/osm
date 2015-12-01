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
ALTER TABLE geometry_columns OWNER TO username;
ALTER TABLE spatial_ref_sys OWNER TO username;
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

Install Haffbuzz:
<pre><code>wget http://www.freedesktop.org/software/harfbuzz/release/harfbuzz-1.1.2.tar.bz2
tar xf harfbuzz-1.1.2.tar.bz2
cd harfbuzz-1.1.2
./configure && make && sudo make install
sudo ldconfig
</code></pre>

Install boost 1.59:
<pre><code>http://downloads.sourceforge.net/boost/boost_1_59_0.tar.bz2
tar xf boost_1_59_0.tar.bz2
cd boost_1_59_0
sed -e '1 i#ifndef Q_MOC_RUN' \
    -e '$ a#endif'            \
    -i boost/type_traits/detail/has_binary_operator.hpp &&
./bootstrap.sh --prefix=/usr &&
./b2 stage threading=multi link=shared
</code></pre>

Build the Mapnik library from source:
<pre><code>git clone git://github.com/mapnik/mapnik
cd maplink
./configure
make && sudo make install
</code></pre>

