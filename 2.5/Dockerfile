FROM ubuntu:xenial
MAINTAINER winsent <pipetc@gmail.com>
RUN apt-get clean && apt-get update && apt-get install -y locales
ENV DEBIAN_FRONTEND noninteractive
ENV LANG C.UTF-8
RUN locale-gen en_US.UTF-8
RUN update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8

# Install packages http://wiki.openstreetmap.org/wiki/Nominatim/Installation#Ubuntu.2FDebian
RUN apt-get install -y build-essential cmake g++ libboost-dev libboost-system-dev \
                        libboost-filesystem-dev libexpat1-dev zlib1g-dev libxml2-dev\
                        libbz2-dev libpq-dev libgeos-dev libgeos++-dev libproj-dev \
                        postgresql-server-dev-9.5 postgresql-9.5-postgis-2.2 postgresql-contrib-9.5 \
                        apache2 php php-pgsql libapache2-mod-php7.0 php-pear curl sudo php-db \
                        php-intl git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* && \
    rm -rf /tmp/* /var/tmp/*

ENV USERNAME=nominatim
ENV USERHOME=/srv/nominatim
RUN mkdir $USERHOME
RUN chmod a+x $USERHOME

WORKDIR /app

# Nominatim install
ENV NOMINATIM_VERSION postcode_support
RUN cd $USERHOME && git clone --recursive https://github.com/PJHaynes304/Nominatim.git
RUN cd $USERHOME/Nominatim && git checkout $NOMINATIM_VERSION && git submodule update --recursive --init && \
  mkdir build && cd build && cmake $USERHOME/Nominatim && make


# Nominatim create site
COPY local.php ./src/settings/local.php
RUN rm -rf /var/www/html/* && mkdir -m 755 /var/www/html/nominatim && chown www-data /var/www/html/nominatim

# Apache configure
COPY nominatim.conf /etc/apache2/conf-available/nominatim.conf

RUN a2enconf nominatim
RUN service apache2 restart

# Load initial data
ARG PBF_DATA=http://download.geofabrik.de/europe/great-britain-latest.osm.pbf
ARG POSTCODE_DATA=http://www.nominatim.org/data/gb_postcode_data.sql.gz
ARG COUNTRY_GRID_DATA=http://www.nominatim.org/data/country_grid.sql.gz
RUN curl -L $PBF_DATA --create-dirs -o /app/src/data.osm.pbf
RUN curl -L $COUNTRY_GRID_DATA --create-dirs -o /srv/nominatim/Nominatim/data/country_osm_grid.sql.gz
RUN curl -L $POSTCODE_DATA -o /srv/nominatim/Nominatim/data/gb_postcode_data.sql.gz
RUN service postgresql start && \
    sudo -u postgres psql postgres -tAc "SELECT 1 FROM pg_roles WHERE rolname='nominatim'" | grep -q 1 || sudo -u postgres createuser -s nominatim && \
    sudo -u postgres psql postgres -tAc "SELECT 1 FROM pg_roles WHERE rolname='www-data'" | grep -q 1 || sudo -u postgres createuser -SDR www-data && \
    sudo -u postgres psql postgres -c "DROP DATABASE IF EXISTS nominatim" && \
    useradd -m -p password1234 nominatim && \
    chown -R nominatim:nominatim $USERHOME/Nominatim && \
    sudo -u nominatim $USERHOME/Nominatim/build/utils/setup.php --osm-file /app/src/data.osm.pbf --all --threads 2 && \
 service postgresql stop

EXPOSE 5432
EXPOSE 8080

COPY start.sh /app/start.sh
CMD /app/start.sh
