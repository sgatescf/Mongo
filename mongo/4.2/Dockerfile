FROM nexus-registry.apps.cloudfitdsop.com/ubi7-golden:7.7
# add our user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
RUN groupadd -r mongodb && useradd --no-log-init -r -g mongodb mongodb

RUN set -eux; \
	yum update --nogpgcheck --disableplugin=subscription-manager -y; \
	yum install --nogpgcheck -y \
		ca-certificates \
		jq \
		numactl;

# grab gosu for easy step-down from root (https://github.com/tianon/gosu/releases)
ENV GOSU_VERSION 1.11

# grab "js-yaml" for parsing mongod's YAML config files (https://github.com/nodeca/js-yaml/releases)
ENV JSYAML_VERSION 3.13.0
ENV ARCH amd64

RUN set -ex; \
	\
	savedAptMark="$(yum list installed)"; \
	yum update; \
	yum install -y --nogpgcheck \
	    wget \
		gnupg \
		dirmngr; \
	\
	wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$ARCH"; \
	wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$ARCH.asc"; \
	export GNUPGHOME="$(mktemp -d)"; \
	gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4; \
	gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu; \
	command -v gpgconf && gpgconf --kill all || :; \
	rm -r "$GNUPGHOME" /usr/local/bin/gosu.asc; \
	chmod +x /usr/local/bin/gosu; \
	gosu --version; \
	gosu nobody true; \
	\
	wget -O /js-yaml.js "https://github.com/nodeca/js-yaml/raw/${JSYAML_VERSION}/dist/js-yaml.js";

    # TODO some sort of download verification here

RUN mkdir /docker-entrypoint-initdb.d

ENV GPG_KEYS E162F504A20CDF15827F718D4B7C549A058F8B6B
RUN set -ex; \
	export GNUPGHOME="$(mktemp -d)"; \
	for key in $GPG_KEYS; do \
		gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys "$key"; \
	done;

# Allow build-time overrides (eg. to build image with MongoDB Enterprise version)
# Options for MONGO_PACKAGE: mongodb-org OR mongodb-enterprise
# Options for MONGO_REPO: repo.mongodb.org OR repo.mongodb.com
# Example: docker build --build-arg MONGO_PACKAGE=mongodb-enterprise --build-arg MONGO_REPO=repo.mongodb.com .
ARG MONGO_PACKAGE=mongodb-org
ARG MONGO_REPO=repo.mongodb.org
ENV MONGO_PACKAGE=${MONGO_PACKAGE} MONGO_REPO=${MONGO_REPO}

ENV MONGO_MAJOR 4.2
ENV MONGO_VERSION 4.2.2

RUN set -x \
	&& yum-config-manager --add-repo=https://repo.mongodb.org/yum/redhat/7Server/$MONGO_PACKAGE/$MONGO_MAJOR/x86_64/ \
	&& yum update --nogpgcheck \
	&& yum install -y --nogpgcheck \
		${MONGO_PACKAGE}-$MONGO_VERSION \
		${MONGO_PACKAGE}-server-$MONGO_VERSION \
		${MONGO_PACKAGE}-shell-$MONGO_VERSION \
		${MONGO_PACKAGE}-mongos-$MONGO_VERSION \
		${MONGO_PACKAGE}-tools-$MONGO_VERSION \
	&& rm -rf /var/lib/apt/lists/* \
	&& rm -rf /var/lib/mongodb \
	&& mv /etc/mongod.conf /etc/mongod.conf.orig

RUN mkdir -p /data/db /data/configdb /home/mongodb \
	&& chown -R mongodb:mongodb /data/db /data/configdb /home/mongodb
VOLUME /data/db /data/configdb

# Disable Ctrl+Alt+Del burst action
RUN echo CtrlAltDelBurstAction=none >> /etc/systemd/system.conf

COPY docker-entrypoint.sh /usr/local/bin/

USER mongodb:mongodb

ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 27017
CMD ["mongod"]
