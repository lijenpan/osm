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
