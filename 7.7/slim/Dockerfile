
FROM openjdk:11-jre-slim

LABEL maintainer="Martijn Koster \"mak-docker@greenhills.co.uk\""
LABEL repository="https://github.com/docker-solr/docker-solr"

# Override the solr download location with e.g.:
#   docker build -t mine --build-arg SOLR_DOWNLOAD_SERVER=http://www-eu.apache.org/dist/lucene/solr .
ARG SOLR_DOWNLOAD_SERVER

RUN apt-get update && \
  apt-get -y install lsof procps wget gpg dirmngr && \
  rm -rf /var/lib/apt/lists/*

ENV SOLR_USER="solr" \
    SOLR_UID="8983" \
    SOLR_GROUP="solr" \
    SOLR_GID="8983" \
    SOLR_VERSION="7.7.0" \
    SOLR_URL="${SOLR_DOWNLOAD_SERVER:-https://archive.apache.org/dist/lucene/solr}/7.7.0/solr-7.7.0.tgz" \
    SOLR_SHA256="ad9eaa29449701fcabad497cc5f0506f8a8f76da714dc4a5429a549fcac24dc7" \
    SOLR_KEYS="052C5B48A480B9CEA9E218A5F98C13CFA5A135D8" \
    PATH="/opt/solr/bin:/opt/docker-solr/scripts:$PATH"

ENV GOSU_VERSION 1.11
ENV GOSU_KEY B42F6819007F00F88E364FD4036A9C25BF357DD4

RUN groupadd -r --gid $SOLR_GID $SOLR_GROUP && \
  useradd -r --uid $SOLR_UID --gid $SOLR_GID $SOLR_USER

RUN set -e; \
  export GNUPGHOME="/tmp/gnupg_home" && \
  mkdir -p "$GNUPGHOME" && \
  chmod 700 "$GNUPGHOME" && \
  echo "disable-ipv6" >> "$GNUPGHOME/dirmngr.conf" && \
  for key in $SOLR_KEYS $GOSU_KEY; do \
    found=''; \
    for server in \
      ha.pool.sks-keyservers.net \
      hkp://keyserver.ubuntu.com:80 \
      hkp://p80.pool.sks-keyservers.net:80 \
      pgp.mit.edu \
    ; do \
      echo "  trying $server for $key"; \
      gpg --batch --keyserver "$server" --keyserver-options timeout=10 --recv-keys "$key" && found=yes && break; \
      gpg --batch --keyserver "$server" --keyserver-options timeout=10 --recv-keys "$key" && found=yes && break; \
    done; \
    test -z "$found" && echo >&2 "error: failed to fetch $key from several disparate servers -- network issues?" && exit 1; \
  done; \
  exit 0

RUN set -e; \
  export GNUPGHOME="/tmp/gnupg_home" && \
  dpkgArch="$(dpkg --print-architecture | awk -F- '{ print $NF }')" && \
  wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch" && \
  wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch.asc" && \
  gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu && \
  rm /usr/local/bin/gosu.asc && \
  chmod +x /usr/local/bin/gosu && \
  gosu nobody true && \
  mkdir -p /opt && \
  echo "downloading $SOLR_URL" && \
  wget -nv $SOLR_URL -O /opt/solr-$SOLR_VERSION.tgz && \
  echo "downloading $SOLR_URL.asc" && \
  wget -nv $SOLR_URL.asc -O /opt/solr-$SOLR_VERSION.tgz.asc && \
  echo "$SOLR_SHA256 */opt/solr-$SOLR_VERSION.tgz" | sha256sum -c - && \
  (>&2 ls -l /opt/solr-$SOLR_VERSION.tgz /opt/solr-$SOLR_VERSION.tgz.asc) && \
  gpg --batch --verify /opt/solr-$SOLR_VERSION.tgz.asc /opt/solr-$SOLR_VERSION.tgz && \
  tar -C /opt --extract --file /opt/solr-$SOLR_VERSION.tgz "solr-$SOLR_VERSION/bin/install_solr_service.sh" --strip-components=2 && \
  /opt/install_solr_service.sh /opt/solr-$SOLR_VERSION.tgz -n && \
  chmod o+r /etc/default/solr.in.sh && \
  rm /opt/install_solr_service.sh /opt/solr-$SOLR_VERSION.tgz* && \
  rm -Rf /opt/solr/docs/ && \
  mkdir -p /opt/solr/server/solr/lib /opt/solr/server/logs /docker-entrypoint-initdb.d /opt/docker-solr && \
  sed -i -e 's/"\$(whoami)" == "root"/$(id -u) == 0/' /opt/solr/bin/solr && \
  sed -i -e 's/lsof -PniTCP:/lsof -t -PniTCP:/' /opt/solr/bin/solr && \
  sed -i -e '/#networkaddress.cache.ttl=-1/ a networkaddress.cache.ttl=10' $(find $(readlink -f $(which java)|sed 's,bin/java,,') -name java.security -print) && \
  chown -R $SOLR_USER:$SOLR_GROUP /opt/solr /var/solr /docker-entrypoint-initdb.d /opt/docker-solr

COPY --chown=solr:solr scripts /opt/docker-solr/scripts

EXPOSE 8983
WORKDIR /opt/solr
USER $SOLR_USER

ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["solr-foreground"]