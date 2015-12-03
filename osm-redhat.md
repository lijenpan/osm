# Preparations
<pre><code>sudo yum groupinstall 'Development Tools'</code></pre>
<pre><code>sudo yum -y install bzip2-devel libpng-devel libtiff-devel zlib-devel libjpeg-devel libxml2-devel python-setuptools proj-devel proj proj-epsg proj-nad freetype-devel freetype libicu-devel libicu gdal-devel gdal sqlite-devel sqlite libcurl-devel libcurl cairo-devel cairo pycairo-devel pycairo geos geos-devel protobuf-devel cmake</code></pre>

# Install PostgreSQL Server and PostGIS
<code>sudo yum install postgresql94-server postgis2_94 postgis2_94-docs postgis2_94-client postgis2_94-utils pgrouting_94</code>

## Initialize Postgresql DB
<code>service postgresql-9.4 initdb</code>

## Start Postgresql Server
<pre><code>chkconfig postgresql-9.4 on
service postgresql-9.4 start
sudo -u postgres createdb gis
</code></pre>

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

### Clone, Build, and Install Mapnik 2.2.0
<pre><code>git clone git://github.com/mapnik/mapnik
cd maplink
./configure ICU_INCLUDES=/usr/local/include ICU_LIBS=/usr/local/include
make && sudo make install</code></pre>

If you experience odd configuration errors, try cleaning the configure caches: <code>make clean</code>
# Install OpenStreetMap
<pre><code>su - postgres
export PATH=$PATH:/usr/pgsql-9.4/bin
psql gis < /usr/pgsql-9.4/share/contrib/postgis-2.1/postgis.sql
psql gis < /usr/pgsql-9.4/share/contrib/postgis-2.1/spatial_ref_sys.sql
createuser osm -W # No to all questions
createuser apache -W # No to all questions
echo "grant all on geometry_columns to apache;" | psql gis
echo "grant all on spatial_ref_sys to apache;" | psql gis
exit

</code></pre>
