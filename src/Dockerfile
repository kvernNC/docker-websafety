# Docker file for running Diladele Web Safety 9.7 on Ubuntu 24.04 (Noble)
FROM phusion/baseimage:noble-1.0.0

# Configuration variables
ENV DEBIAN_FRONTEND=noninteractive \
    TERM=xterm \
    DDWS_VER=9.7.0.7FA4 \
    DDWS_UI_VER=9.7.0.8C8A \
    OSNAME=ubuntu24 \
    WEBSAFETY_IN_DOCKER="True"

# --- BLOC 1 : Systèmes, Dépendances Python et Apache ---
# On installe tout en une fois pour garder l'index apt valide
RUN apt-get update && apt-get upgrade -y -o Dpkg::Options::="--force-confold" && \
    apt-get install -y --no-install-recommends \
    apt-utils apache2 libapache2-mod-wsgi-py3 iproute2 sudo curl gnupg \
    python3-pip python3-dev python3-setuptools python3-venv python3.12-venv \
    libsasl2-dev libldap2-dev libssl-dev g++ krb5-user sqlite3 \
    libjpeg-dev zlib1g-dev cron htop mc net-tools jq vim \
    # Paquets Django/Python officiels du dépôt Noble
    python3-django python3-tz python3-requests python3-pandas python3-yaml \
    python3-psutil python3-tld python3-jinja2 python3-openssl \
    python3-cryptography python3-google-auth python3-ldap && \
    # Nettoyage des listes APT
    apt-get clean && rm -rf /var/lib/apt/lists/*

# --- BLOC 2 : Squid natif (Noble utilise Squid 6.x par défaut) ---
RUN apt-get update && apt-get install -y --no-install-recommends \
    squid-common squid-openssl squidclient && \
    rm -rf /var/run/squid.pid && \
    # Initialisation de la DB SSL pour l'interception HTTPS
    SSL_DB=/var/spool/squid/ssldb && \
    rm -rf $SSL_DB && \
    /usr/lib/squid/security_file_certgen -c -s $SSL_DB -M 4MB && \
    chown -R proxy:proxy $SSL_DB /var/log/squid && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# --- BLOC 3 : Installation Web Safety Core & UI ---
RUN curl -fSL https://www.diladele.com/pkg/websafety-core/${DDWS_VER}/amd64/release/${OSNAME}/websafety-${DDWS_VER}_amd64.deb -o core.deb && \
    curl -fSL https://www.diladele.com/pkg/websafety-ui/${DDWS_UI_VER}/amd64/release/${OSNAME}/websafety-ui-${DDWS_UI_VER}_amd64.deb -o ui.deb && \
    apt-get update && \
    apt-get install -y ./core.deb ./ui.deb && \
    rm -f core.deb ui.deb && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# --- BLOC 4 : Virtual Environment & Génération (03_venv.sh) ---
RUN python3 -m venv /opt/websafety-ui/env && \
    /opt/websafety-ui/env/bin/pip install --upgrade pip wheel && \
    /opt/websafety-ui/env/bin/pip install -r /opt/websafety-ui/var/console/requirements.txt && \
    # Préparation des dossiers et droits avant génération
    mkdir -p /opt/websafety-ui/var/db /opt/websafety-ui/var/logs /opt/websafety/var/reports && \
    chown -R websafety:websafety /opt/websafety-ui && \
    chown -R proxy:proxy /opt/websafety && \
    # Génération via le python du venv
    sudo -u proxy /opt/websafety-ui/env/bin/python3 /opt/websafety-ui/var/console/generate.py --core && \
    sudo -u websafety /opt/websafety-ui/env/bin/python3 /opt/websafety-ui/var/console/generate.py --ui

# --- BLOC 5 : Intégration Apache (04_integrate.sh) ---
RUN a2enmod ssl && \
    a2dissite 000-default && \
    a2ensite websafety && \
    mkdir -p /var/run/apache2 /var/log/websafety && \
    # Lien symbolique "force" pour corriger le bug de path de Squid
    mkdir -p /usr/share/squid/errors/templates/opt/websafety/etc/squid && \
    ln -sf /opt/websafety/etc/squid/portal.html /usr/share/squid/errors/templates/opt/websafety/etc/squid/portal.html && \
    touch /etc/crontab

# Sudoers & Scripts
RUN echo "websafety ALL=(root) NOPASSWD: /opt/websafety/bin/reload.sh, /opt/websafety/bin/restart.sh, /opt/websafety/bin/cluster.sh, /opt/websafety-ui/bin/certmgr.py, /opt/websafety-ui/bin/cachemgr.py, /usr/bin/tail, /opt/websafety-ui/bin/timezone.py, /opt/websafety-ui/bin/hostname.py, /opt/websafety-ui/bin/resolve.py, /opt/websafety-ui/bin/netmgr.py, /opt/websafety-ui/bin/fsmgr.py, /opt/websafety-ui/bin/cronmgr.py, /opt/websafety-ui/bin/logrotatemgr.py, /opt/websafety-ui/bin/confmgr.py" >> /etc/sudoers

# Copy required files
COPY src/contents/squid.conf /etc/squid/squid.conf
COPY src/contents/websafety_update /etc/cron.daily
COPY src/contents/websafety_license /etc/cron.daily
COPY src/contents/toulousecategories /etc/cron.daily
COPY src/contents/wsicapd /etc/service/wsicapd/run
COPY src/contents/wsgsbd /etc/service/wsgsbd/run
COPY src/contents/wsytgd /etc/service/wsytgd/run
COPY src/contents/wssyncd /etc/service/wssyncd/run
COPY src/contents/wsmond /etc/service/wsmond/run
COPY src/contents/squid /etc/service/squid/run
COPY src/contents/apache2 /etc/service/apache2/run
COPY src/contents/reload.sh /opt/websafety/bin/reload.sh
COPY src/contents/restart.sh /opt/websafety/bin/restart.sh

# FIX permissions
RUN chmod +x /opt/websafety/bin/* && \
    chmod +x /etc/cron.daily/* && \
    chmod u+s /opt/websafety/bin/* && \
    chmod +x /etc/service/squid/run \
             /etc/service/wsicapd/run \
             /etc/service/wsgsbd/run \
             /etc/service/wsytgd/run \
             /etc/service/wsmond/run \
             /etc/service/wssyncd/run \
             /etc/service/apache2/run

# --- FIX FINAL DES PERMISSIONS ---
# On s'assure que l'utilisateur websafety possède ses dossiers de travail
RUN chown -R websafety:websafety /opt/websafety/var && \
    chown -R websafety:websafety /opt/websafety-ui/var && \
    # On nettoie les résidus de dossiers supervise qui pourraient être dans le repo git
    find /etc/service -name "supervise" -type d -exec rm -rf {} + && \
    # On s'assure que runit peut créer ses fichiers de contrôle
    chmod -R 755 /etc/service            
             
#FIX missing sqlite
RUN mkdir -p /opt/websafety-ui/var/stats /opt/websafety-ui/var/reports
RUN sqlite3 /opt/websafety-ui/var/stats/realtime.sqlite "VACUUM;"
RUN chown -R websafety:websafety /opt/websafety-ui/var/stats /opt/websafety-ui/var/reports && \
    chmod -R 775 /opt/websafety-ui/var/stats

#FIX missing files
RUN touch /opt/websafety/var/log/cron_update.log
RUN touch /opt/websafety/var/log/wssyncd.log
RUN touch /opt/websafety-ui/var/log/wsmond.log
RUN touch /opt/websafety/var/log/wsicapd.log
RUN touch /opt/websafety/var/log/wsytgd.log

CMD ["/sbin/my_init"]