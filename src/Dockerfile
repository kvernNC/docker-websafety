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
RUN apt-get update && apt-get upgrade -y -o Dpkg::Options::="--force-confold" && \
    apt-get install -y --no-install-recommends \
    apt-utils apache2 libapache2-mod-wsgi-py3 iproute2 sudo curl gnupg \
    python3-pip python3-dev python3-setuptools python3-venv python3.12-venv \
    libsasl2-dev libldap2-dev libssl-dev g++ krb5-user sqlite3 \
    libjpeg-dev zlib1g-dev cron htop mc net-tools jq vim \
    python3-django python3-tz python3-requests python3-pandas python3-yaml \
    python3-psutil python3-tld python3-jinja2 python3-openssl \
    python3-cryptography python3-google-auth python3-ldap3 python3-certifi

# --- BLOC 2 : Installation Web Safety (Core & UI) ---
RUN curl https://www.diladele.com/pkg/websafety-core/$DDWS_VER/amd64/$OSNAME/websafety-$DDWS_VER-amd64.deb -o websafety.deb && \
    dpkg -i websafety.deb && rm websafety.deb && \
    curl https://www.diladele.com/pkg/websafety-ui/$DDWS_UI_VER/amd64/$OSNAME/websafety-ui-$DDWS_UI_VER-amd64.deb -o websafety-ui.deb && \
    dpkg -i websafety-ui.deb && rm websafety-ui.deb

# --- BLOC 3 : Configuration Apache ---
RUN a2dissite 000-default && \
    ln -s /opt/websafety-ui/etc/apache2/websafety-ui.conf /etc/apache2/sites-available/websafety-ui.conf && \
    a2ensite websafety-ui.conf && \
    # On permet à apache d'écrire dans son dossier de logs pour éviter les erreurs au reload
    chown -R www-data:www-data /var/log/apache2

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

# --- BLOC 5 : Préparation des données et SQLite ---
# On crée les dossiers AVANT de fixer les permissions
RUN mkdir -p /opt/websafety-ui/var/stats /opt/websafety-ui/var/reports /opt/websafety/var/log /opt/websafety/var/db && \
    sqlite3 /opt/websafety-ui/var/stats/realtime.sqlite "VACUUM;" && \
    touch /opt/websafety/var/log/cron_update.log /opt/websafety/var/log/wssyncd.log

# --- BLOC 6 : FIX FINAL DES PERMISSIONS (Indispensable pour Kubernetes) ---
# On applique les droits globalement
RUN chown -R websafety:websafety /opt/websafety && \
    chown -R websafety:websafety /opt/websafety-ui && \
    # SUID sur les binaires pour qu'ils puissent binder les ports/fichiers système
    chmod u+s /opt/websafety/bin/* && \
    chmod +x /opt/websafety/bin/*.sh && \
    chmod +x /etc/cron.daily/* && \
    # Nettoyage runit pour forcer une réinitialisation propre au démarrage
    find /etc/service -name "supervise" -type d -exec rm -rf {} + && \
    chmod -R 755 /etc/service && \
    chmod +x /etc/service/*/run

# Utilisation de l'init système de Phusion
CMD ["/sbin/my_init"]