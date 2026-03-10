#
# Docker file for running Diladele Web Safety in one container
#
FROM phusion/baseimage:noble-1.0.0

#MAINTAINER support@diladele.com

# set configuration variables
ENV DEBIAN_FRONTEND noninteractive
ENV TERM xterm
ENV DDWS_VER=9.7.0.7FA4
ENV DDWS_UI_VER=9.7.0.8C8A
ENV OSNAME=ubuntu24
ENV DEBIAN_FRONTEND=noninteractive
ENV WEBSAFETY_IN_DOCKER "True"

# install python libs and apache with modwsgi
RUN apt-get update && apt-get upgrade -y -o Dpkg::Options::="--force-confold" && \
    apt-get install -y --no-install-recommends \
    apt-utils apache2 libapache2-mod-wsgi-py3 iproute2 \
    python3-pip python3-dev python3-setuptools python3-venv \
    sudo curl gnupg libsasl2-dev libldap2-dev libssl-dev g++ \
    krb5-user sqlite3 && \
    a2dissite 000-default && a2enmod ssl && \
    curl https://packages.diladele.com/diladele_pub.asc -s -o diladele_pub.asc && \
    APT_KEY_DONT_WARN_ON_DANGEROUS_USAGE=1 apt-key add diladele_pub.asc && rm diladele_pub.asc && \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
#    python3 -m pip install --upgrade pip --break-system-packages && \
#    python3 -m pip install wheel --break-system-packages && \
#    python3 -m pip install --break-system-packages \
#        django==4.1.2 \
#        pytz \
#        requests \
#        pandas \
#        PyYAML \
#        psutil \
#        tld \
#        jinja2 \
#        msal pyOpenSSL cryptography google-auth google-auth-oauthlib && \
#    python3 -m pip uninstall -y ldap ldap3 python-ldap || true && \
#    python3 -m pip install python-ldap --break-system-packages
    apt-get install -y --no-install-recommends \
    python3-django \
    python3-tz \
    python3-requests \
    python3-pandas \
    python3-yaml \
    python3-psutil \
    python3-tld \
    python3-jinja2 \
    python3-openssl \
    python3-cryptography \
    python3-google-auth \
    python3-ldap \
    python3-dev \
    python3.12-venv \
    libjpeg-dev \
    zlib1g-dev

RUN echo "deb https://squid57.diladele.com/ubuntu/ focal main" > /etc/apt/sources.list.d/squid57.diladele.com.list && \
    apt-get update && apt-get install -y --no-install-recommends \
            libecap3 libecap3-dev \
            squid-common \
            squid-openssl \
            squidclient && \
    rm -rf /var/run/squid.pid && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# get latest web safety and install it
#RUN curl https://www.diladele.com/pkg/websafety-core/${DDWS_VER}/amd64/release/${OSNAME}/websafety-${DDWS_VER}_amd64.deb -o websafety-core${DDWS_VER}_amd64.deb && \
#    curl https://www.diladele.com/pkg/websafety-ui/${DDWS_UI_VER}/amd64/release/${OSNAME}/websafety-ui-${DDWS_UI_VER}_amd64.deb -o websafety-ui-${DDWS_UI_VER}_amd64.deb && \
#    dpkg --install websafety-core${DDWS_VER}_amd64.deb && \
#    dpkg --install websafety-ui-${DDWS_UI_VER}_amd64.deb && \
#    chown -R proxy:proxy /opt/websafety && \
#    chown -R websafety:websafety /opt/websafety-ui && \
#    rm -f websafety-core${DDWS_VER}_amd64.deb && \
#    rm -f websafety-ui-${DDWS_UI_VER}_amd64.deb && \
#    a2ensite websafety && \
#    mkdir -p /var/run/apache2 && \
#    mkdir -p /var/log/websafety && \
#    sudo -u proxy python3 /opt/websafety-ui/var/console/generate.py --core && \
#    sudo -u websafety python3 /opt/websafety-ui/var/console/generate.py --ui && \
#    chown -R websafety:websafety /opt/websafety-ui && \
#    touch /etc/crontab && mkdir -p /opt/websafety/var/reports && \
#    mkdir -p /usr/share/squid/errors/templates/opt/websafety/etc/squid && \
#    ln -s /opt/websafety/etc/squid/portal.html /usr/share/squid/errors/templates/opt/websafety/etc/squid/portal.html

# Bloc 1: Download & Install
RUN curl -fSL https://www.diladele.com/pkg/websafety-core/${DDWS_VER}/amd64/release/${OSNAME}/websafety-${DDWS_VER}_amd64.deb -o core.deb && \
    curl -fSL https://www.diladele.com/pkg/websafety-ui/${DDWS_UI_VER}/amd64/release/${OSNAME}/websafety-ui-${DDWS_UI_VER}_amd64.deb -o ui.deb && \
    apt-get update && \
    apt-get install -y ./core.deb ./ui.deb && \
    rm -f core.deb ui.deb

# Bloc 2: Python Virtual Environment (03_venv.sh)
RUN python3 -m venv /opt/websafety-ui/env && \
    /opt/websafety-ui/env/bin/pip install --upgrade pip wheel && \
    /opt/websafety-ui/env/bin/pip install -r /opt/websafety-ui/var/console/requirements.txt && \
    # Préparation des dossiers pour éviter les erreurs de permissions
    mkdir -p /opt/websafety-ui/var/db /opt/websafety-ui/var/logs /opt/websafety/var/reports && \
    chown -R websafety:websafety /opt/websafety-ui && \
    chown -R proxy:proxy /opt/websafety && \
    # Génération des fichiers de base via le venv
    sudo -u proxy /opt/websafety-ui/env/bin/python3 /opt/websafety-ui/var/console/generate.py --core && \
    sudo -u websafety /opt/websafety-ui/env/bin/python3 /opt/websafety-ui/var/console/generate.py --ui

# Bloc 3: Apache Integration & Final Links (04_integrate.sh)
RUN a2enmod ssl && \
    a2dissite 000-default && \
    a2ensite websafety && \
    mkdir -p /var/run/apache2 /var/log/websafety && \
    # On s'assure que le lien symbolique écrase l'existant (-sf)
    mkdir -p /usr/share/squid/errors/templates/opt/websafety/etc/squid && \
    ln -sf /opt/websafety/etc/squid/portal.html /usr/share/squid/errors/templates/opt/websafety/etc/squid/portal.html && \
    touch /etc/crontab

RUN echo "websafety ALL=(root) NOPASSWD: /opt/websafety/bin/reload.sh, /opt/websafety/bin/restart.sh, /opt/websafety/bin/cluster.sh, /opt/websafety-ui/bin/certmgr.py, /opt/websafety-ui/bin/cachemgr.py, /usr/bin/tail, /opt/websafety-ui/bin/timezone.py, /opt/websafety-ui/bin/hostname.py, /opt/websafety-ui/bin/resolve.py, /opt/websafety-ui/bin/netmgr.py, /opt/websafety-ui/bin/fsmgr.py, /opt/websafety-ui/bin/cronmgr.py, /opt/websafety-ui/bin/logrotatemgr.py, /opt/websafety-ui/bin/confmgr.py" >> /etc/sudoers

# copy required files
COPY src/contents/squid.conf /etc/squid/squid.conf
COPY src/contents/websafety_update /etc/cron.daily
COPY src/contents/websafety_license /etc/cron.daily
COPY src/contents/wsicapd /etc/service/wsicapd/run
COPY src/contents/wsgsbd /etc/service/wsgsbd/run
COPY src/contents/wsytgd /etc/service/wsytgd/run
COPY src/contents/wssyncd /etc/service/wssyncd/run
COPY src/contents/wsmond /etc/service/wsmond/run
COPY src/contents/squid /etc/service/squid/run
COPY src/contents/apache2 /etc/service/apache2/run
COPY src/contents/reload.sh /opt/websafety/bin/reload.sh
COPY src/contents/restart.sh /opt/websafety/bin/restart.sh

# reset owner of installation path
RUN chmod +x /opt/websafety/bin/* && \
    chmod +x /etc/cron.daily/websafety_update && \
    chmod u+s /opt/websafety/bin/* && \
    chmod +x /etc/service/squid/run \
             /etc/service/wsicapd/run \
             /etc/service/wsgsbd/run \
             /etc/service/wsytgd/run \
             /etc/service/wsmond/run \
             /etc/service/apache2/run

# assign volumes
#VOLUME ["/opt/websafety/etc"]
#VOLUME ["/opt/websafety/var/spool"]
#VOLUME ["/opt/websafety/var/console"]
#VOLUME ["/opt/websafety/var/db"]
#VOLUME ["/opt/websafety/var/log"]
#VOLUME ["/opt/websafety/var/reports"]
#VOLUME ["/etc/squid"]
#VOLUME ["/var/spool"]
#VOLUME ["/var/log/squid"]

CMD ["/sbin/my_init"]
