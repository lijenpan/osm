# Preparations
<pre><code>sudo yum groupinstall 'Development Tools'</code></pre>
<pre><code>sudo yum -y install bzip2-devel libpng-devel libtiff-devel zlib-devel libjpeg-devel libxml2-devel python-setuptools proj-devel proj proj-epsg proj-nad freetype-devel freetype libicu-devel libicu gdal-devel gdal sqlite-devel sqlite libcurl-devel libcurl cairo-devel cairo pycairo-devel pycairo geos geos-devel protobuf-devel</code></pre>

# Install PostgreSQL Server and PostGIS
<code>sudo yum install postgis2_94 postgis2_94-docs postgis2_94-client postgis2_94-utils pgrouting_94</code>

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

### Clone, Build, and Install Mapnik
<pre><code>git clone git://github.com/mapnik/mapnik
cd maplink
./configure
make && sudo make install</code></pre>

# Install OpenStreetMap
