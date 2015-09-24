# This is a Dockerfile to build a erlang runtime image.
FROM dasudian/docker-erlang:17.5
MAINTAINER Mengz You <mengz.you@outlook.com>

# Environments
ENV MONGOOSEIM_HOME /opt/mongooseim
ENV MONGOOSEIM_VERSION 1.5.1
ENV MONGOOSEIM_REL_DIR /opt/mongooseim/rel/mongooseim
ENV PATH /opt/mongooseim/rel/mongooseim/bin/:$PATH

# Update and install build tool packages
RUN apt-get update \
  && apt-get install -y --no-install-recommends git libexpat1-dev libpam0g-dev libxml2-dev \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*

# Download MongooseIM source codes
RUN wget -O /opt/MongooseIM.tar.gz https://github.com/esl/MongooseIM/archive/1.5.1.tar.gz \
  && cd /opt/ && tar -zxf MongooseIM.tar.gz && mv MongooseIM-1.5.1 mongooseim \
  && cd mongooseim/ \
  && make configure with-pgsql with-redis && make rel \
  && cd / && rm -rf /opt/MongooseIM.tar.gz \
  && mkdir -p /data/{log,mnesia} \
  && rm -rf /opt/mongooseim/rel/mongooseim/log \
  && ln -s /data/log /opt/mongooseim/rel/mongooseim/log

ADD ./docker-entry.sh docker-entry.sh

# expose xmpp, rest, s2s, epmd, distributed erlang
EXPOSE 5222 5280 5269 4369 9100

# Define mount points.
VOLUME ["/data/mnesia", "/data/log"]

ENTRYPOINT ["/docker-entry.sh"]
