# Preparations
<pre><code>sudo yum groupinstall 'Development Tools'</code></pre>
<pre><code>sudo yum install pycairo.x86_64 pycairo-devel.x86_64 geos geos-devel protobuf-devel</code></pre>

Download, compile and install proj.4: https://github.com/OSGeo/proj.4/tree/4.9.2

# Install PostgreSQL Server and PostGIS
<code>sudo yum install postgis2_94.x86_64 postgis2_94-docs.x86_64 postgis2_94-client.x86_64 postgis2_94-utils.x86_64 pgrouting_94.x86_64</code>

# Build and Install Mapnik
## Boost C++ Libraries
<pre><code>wget http://downloads.sourceforge.net/boost/boost_1_59_0.tar.bz2
tar xf boost_1_59_0.tar.bz2
cd boost_1_59_0
./bootstrap.sh
./b2 isntall</code></pre>

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
