# Docker file for running Diladele Web Safety 9.7 on Ubuntu 24.04 (Noble)
FROM phusion/baseimage:noble-1.0.0

# Configuration variables
ENV DEBIAN_FRONTEND=noninteractive \
    TERM=xterm \
    DDWS_VER=9.7.0.7FA4 \
    DDWS_UI_VER=9.7.0.8C8A \
    OSNAME=ubuntu24 \
    WEBSAFETY_IN_DOCKER="True" \
    DEBIAN_FRONTEND=noninteractive

# --- BLOC 1 : Systèmes, Dépendances Python et Apache ---
RUN apt-get update && apt-get upgrade -y -o Dpkg::Options::="--force-confold" && \
    apt-get install -y --no-install-recommends \
    apt-utils apache2 libapache2-mod-wsgi-py3 iproute2 sudo curl gnupg \
    python3-pip python3-dev python3-setuptools python3-venv python3.12-venv \
    libsasl2-dev libldap2-dev libssl-dev g++ krb5-user sqlite3 \
    libjpeg-dev zlib1g-dev cron htop mc net-tools jq vim \
    python3-django python3-tz python3-requests python3-pandas python3-yaml \
    python3-psutil python3-tld python3-jinja2 python3-openssl \
    python3-cryptography python3-google-auth python3-ldap3 python3-certifi\
    cron htop mc net-tools jq vim krb5-user python3-ldap

# --- BLOC 2 : Follow manual install docs ---
RUN curl https://www.diladele.com/pkg/websafety-core/${DDWS_VER}/amd64/release/${OSNAME}/websafety-${DDWS_VER}_amd64.deb -o websafety.deb && \
    dpkg -i websafety.deb && rm websafety.deb && \
    curl https://www.diladele.com/pkg/websafety-ui/${DDWS_UI_VER}/amd64/release/${OSNAME}/websafety-ui-${DDWS_UI_VER}_amd64.deb -o websafety-ui.deb && \
    dpkg -i websafety-ui.deb && rm websafety-ui.deb && \
    curl https://www.diladele.com/pkg/diladele_pub.asc -o diladele_pub.asc && \
    cat diladele_pub.asc | gpg --dearmor -o diladele_pub.asc.gpg && \
    mv diladele_pub.asc.gpg /etc/apt/trusted.gpg.d/ && \
    echo "deb https://diladele.github.io/repo-squid-7_2_2-ubuntu-24_04/repo/ubuntu/ noble main" \
    > /etc/apt/sources.list.d/squid-7_2_2.diladele.github.io.list && \
    apt update && apt install -y --no-install-recommends  \
    squid-common \
    squid-openssl \
    libecap3 libecap3-dev && \
    mkdir -p /etc/systemd/system/squid.service.d && \
    echo "[Service]"         >> /etc/systemd/system/squid.service.d/override.conf && \
    echo "LimitNOFILE=65535" >> /etc/systemd/system/squid.service.d/override.conf && \
    update-alternatives --set squid /usr/sbin/squid-openssl && \
    a2enmod ssl && \
    a2dissite 000-default && \
    a2ensite websafety && \
    mkdir -p /usr/share/squid/errors/templates/opt/websafety/etc/squid && \
    ln -s /opt/websafety/etc/squid/portal.html /usr/share/squid/errors/templates/opt/websafety/etc/squid/portal.html && \
    /usr/lib/squid/security_file_certgen -c -s /var/spool/squid/ssldb -M 4MB && \
    chown -R proxy:proxy /var/spool/squid/ssldb && \
    python3 -m venv /opt/websafety-ui/env && \
    /opt/websafety-ui/env/bin/pip install --upgrade pip wheel && \
    /opt/websafety-ui/env/bin/pip install -r /opt/websafety-ui/var/console/requirements.txt && \
    mkdir -p /opt/websafety-ui/var/db /opt/websafety-ui/var/logs /opt/websafety/var/reports && \
    chown -R websafety:websafety /opt/websafety-ui && \
    chown -R proxy:proxy /opt/websafety && \
    /opt/websafety-ui/env/bin/pip3 install -r /opt/websafety-ui/var/console/requirements.txt && \
    sudo -u proxy /opt/websafety-ui/env/bin/python3 /opt/websafety-ui/var/console/generate.py --core && \
    sudo -u websafety /opt/websafety-ui/env/bin/python3 /opt/websafety-ui/var/console/generate.py --ui 

# --- BLOC 4 : Scripts et Services (Runit & Cron) ---
# Note : Tes fichiers run sont maintenant "propres" (sans chown interne)
COPY src/contents/websafety_update /etc/cron.daily/websafety_update
COPY src/contents/toulousecategories /etc/cron.daily/toulousecategories
COPY src/contents/wsicapd /etc/service/wsicapd/run
COPY src/contents/wsgsbd /etc/service/wsgsbd/run
COPY src/contents/wsytgd /etc/service/wsytgd/run
COPY src/contents/wssyncd /etc/service/wssyncd/run
COPY src/contents/wsmond /etc/service/wsmond/run
COPY src/contents/squid /etc/service/squid/run
COPY src/contents/apache2 /etc/service/apache2/run
COPY src/contents/reload.sh /opt/websafety/bin/reload.sh
COPY src/contents/restart.sh /opt/websafety/bin/restart.sh
COPY src/contents/squid.conf /etc/squid/squid.conf
COPY src/contents/updateatboot /etc/init.d/update


# --- BLOC 5 : Préparation des données et SQLite ---
# On crée les dossiers AVANT de fixer les permissions
RUN mkdir -p /opt/websafety-ui/var/stats /opt/websafety-ui/var/reports /opt/websafety/var/log /opt/websafety/var/db && \
    sqlite3 /opt/websafety-ui/var/stats/realtime.sqlite "VACUUM;" && \
    touch /opt/websafety/var/log/cron_update.log /opt/websafety/var/log/wssyncd.log

# --- BLOC 6 : FIX FINAL DES PERMISSIONS (Indispensable pour Kubernetes) ---
# On applique les droits globalement
RUN chown -R proxy:proxy /opt/websafety && \
    chown -R websafety:websafety /opt/websafety-ui && \
    # DOCS
    chown -R proxy:proxy /var/spool/squid/ssldb && \
    chown -R proxy:proxy /var/log/squid && \
    # SUID sur les binaires pour qu'ils puissent binder les ports/fichiers système
    chmod u+s /opt/websafety/bin/* && \
    chmod +x /opt/websafety/bin/*.sh && \
    chmod +x /etc/cron.daily/* && \
    # Nettoyage runit pour forcer une réinitialisation propre au démarrage
    find /etc/service -name "supervise" -type d -exec rm -rf {} + && \
    chmod -R 755 /etc/service && \
    chmod +x /etc/service/*/run && \
    chmod +x /etc/init.d/update

# Utilisation de l'init système de Phusion
CMD ["/sbin/my_init"]