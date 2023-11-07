FROM --platform=linux/amd64 debian:bookworm-slim as base

USER root

SHELL ["/bin/bash", "-euo", "pipefail", "-c"]
ENV SHELL=/bin/bash

RUN apt-get update && \
    apt-get upgrade --yes && \
    apt-get install --yes \
    gettext-base \
    locales \
    python3 \
    python3-pip \
    python3-venv \
    git-lfs \
    unzip \
    neovim \
    openjdk-17-jdk \
    maven \
    ca-certificates-java \
    ca-certificates \
    zip && \
    rm -rf /var/lib/apt/lists/*

RUN echo "en_GB.UTF-8 UTF-8" >> /etc/locale.gen && \
    locale-gen en_GB.UTF-8 && \
    update-locale en_GB.UTF-8 && \
    dpkg-reconfigure --frontend=noninteractive locales && \
    echo "LANG=en_GB.UTF-8" >> /etc/default/locale && \
    echo "LANGUAGE=en_GB:en" >> /etc/default/locale && \
    echo "LC_ALL=en_GB.UTF-8" >> /etc/default/locale
ENV LANG="en_GB.UTF-8"
ENV LANGUAGE="en_GB:en"
ENV LC_ALL="en_GB.UTF-8"

# Create techuser with UID
ARG UID=1001
RUN useradd -l -m -u $UID techuser && \
    echo "techuser:tmp_passwd" | chpasswd \
    && chown techuser /home/techuser
ENV HOME=/home/techuser

# This in analogous to the virtualenv activate script
# To allow deactivation of the virtualenv, we need to save the old PATH
ENV _OLD_VIRTUAL_PATH="$PATH"
ENV VIRTUAL_ENV=/opt/.venv
ENV PATH="$VIRTUAL_ENV/bin:$PATH"

COPY --chmod=755 hooks/* /opt/git/global-hooks/

WORKDIR /opt/git/global-hooks

RUN ln -s "$(which python3.11)" /usr/bin/python && \
    ln -sf "$(which python3.11)" /usr/bin/python3 && \
    ln -sf "$(which pip3.11)" /usr/local/bin/pip && \
    ln -sf "$(which pip3.11)" /usr/local/bin/pip3 && \
    python -m venv /opt/.venv && \
    # Configure pre-commit
    pip install pre-commit==3.4.0 --no-cache-dir && \
    echo "commit-msg post-rewrite pre-commit pre-merge-commit pre-rebase prepare-commit-msg" | xargs -n 1 cp /opt/git/global-hooks/+pre-commit-only.sh && \
    echo "pre-push post-checkout post-commit post-merge" | xargs -n 1 cp /opt/git/global-hooks/+pre-commit-and-lfs.sh && \
    git config --global core.hooksPath /opt/git/global-hooks && \
    chmod -R 755 /opt/git/global-hooks && \
    chown -R techuser /opt/.venv/bin/ /opt/.venv/lib/python3.11/site-packages

# Make pre-commit cache persistent
ENV PRE_COMMIT_HOME=/workspace/.pre-commit

WORKDIR /workspace

##
# Подключить образ Капеллы, основанный на базовом
FROM --platform=linux/amd64 base as capella

ENV DEBIAN_FRONTEND=noninteractive

SHELL ["/bin/bash", "-euo", "pipefail", "-c"]
ENV SHELL=/bin/bash

USER root
COPY libs /tmp/libs
RUN apt-get update && \
    apt-get install -y \
    libjavascriptcoregtk-4.0-18 \
    libwebkit2gtk-4.0-37 && \
    rm -rf /var/lib/apt/lists/*;

WORKDIR /opt/

# Скачиваем Капеллу (вариант установки "Онлайн")
ARG CAPELLA_VERSION
COPY ./download_archive.py /opt/.download_archive.py
RUN pip install --no-cache-dir requests==2.31.0 lxml==4.9.3 && \
    python .download_archive.py ${CAPELLA_VERSION};

RUN apt-get update && \
    apt-get install -y \
    libxtst6 \
    xdg-utils \
    xvfb \
    xauth \
    dbus-x11 && \
    rm -rf /var/lib/apt/lists/*;

RUN if [ -s capella.zip ]; then \
    # Install Capella using zip
    unzip capella.zip -d . && \
    rm capella.zip; \
    # Install Capella using tar.gz
    elif [ -s capella.tar.gz ]; then \
    tar -xf capella.tar.gz; \
    else \
    echo "capella.zip and capella.tar.gz are empty! Please add you custom Capella zip or tar.gz" && \
    exit 1; \
    fi && \
    rm -rf samples capella.zip capella.tar.gz

# Set Permissions
RUN chmod +x capella/capella && \
    chmod -R +x capella/jre/bin && \
    chmod -R +x capella/jre/lib && \
    chown -R techuser /opt/capella

COPY patch.sh /opt/patch.sh
RUN chmod +x /opt/patch.sh

# Install Dropins for Capella
USER techuser

COPY ./versions/${CAPELLA_VERSION}/dropins /opt/capella/dropins
ARG CAPELLA_DROPINS=""
COPY install_dropins.py /opt/install_dropins.py
COPY ./versions/${CAPELLA_VERSION}/dropins.yml /opt/dropins.yml

ARG MEMORY_LIMIT=5500m

RUN echo '-Dorg.eclipse.equinox.p2.transport.ecf.retry=15' >> /opt/capella/capella.ini && \
    echo '-Dorg.eclipse.ecf.provider.filetransfer.retrieve.readTimeout=10000' >> /opt/capella/capella.ini && \
    sed -i "s/-Xmx[^ ]*/-Xmx$MEMORY_LIMIT/g" /opt/capella/capella.ini
RUN pip install --no-cache-dir PyYAML==6.0.1 && python install_dropins.py

COPY ./versions/${CAPELLA_VERSION}/patches /opt/patches
RUN PATCH_DIR=/opt/patches /opt/patch.sh

USER root

# Eclipse settings
RUN mkdir -p /opt/capella/configuration/.settings; \
    ## Do not show WORKSPACE_SELECTION_DIALOG
    echo "SHOW_WORKSPACE_SELECTION_DIALOG=false" >> /opt/capella/configuration/.settings/org.eclipse.ui.ide.prefs; \
    # Setup workspace
    mkdir -p /workspace; \
    ## Disable Welcome Screen
    mkdir -p /workspace/.metadata/.plugins/org.eclipse.core.runtime/.settings/org.eclipse.ui.prefs; \
    # Set workspace permissions
    chown -R techuser /workspace && \
    chmod -R 777 /opt/capella/configuration && \
    chmod -R 777 /opt/capella/p2/org.eclipse.equinox.p2.engine/profileRegistry && \
    chown techuser /opt /opt/capella/capella.ini

RUN echo '-Dosgi.configuration.area=file:/opt/capella/configuration' >> /opt/capella/capella.ini
COPY setup_workspace.py /opt/setup/setup_welcome_screen.py
COPY replace_env_variables.sh /opt/setup/replace_env_variables.sh

ENV AUTOSTART_CAPELLA=1
ENV RESTART_CAPELLA=1
COPY ./autostart /home/techuser/.config/openbox/autostart

ENV ECLIPSE_INSTALLATION_PATH=/opt/capella
ENV ECLIPSE_EXECUTABLE=/opt/capella/capella

COPY ./autostart /home/techuser/.config/openbox/autostart

ARG BUILD_ARCHITECTURE=amd64
ADD https://github.com/krallin/tini/releases/download/v0.19.0/tini-$BUILD_ARCHITECTURE /tini
RUN chmod +x /tini

COPY startup.sh /startup.sh
ENTRYPOINT [ "/tini", "--", "/startup.sh" ]

ENV BASE_TYPE=capella

##
# Подключить remote, расширяющий Капеллу
FROM capella as remote

ARG DEBIAN_FRONTEND=noninteractive

SHELL ["/bin/bash", "-euo", "pipefail", "-c"]
ENV SHELL=/bin/bash

# Install RDP (XRDP with XORG)
USER root

RUN apt-get update && apt-get upgrade -y && apt-get install -y \
    xrdp \
    xserver-xorg-core \
    xorgxrdp \
    obconf \
    gettext-base \
    xprintidle \
    nitrogen && rm -rf /var/lib/apt/lists/*

COPY requirements.txt ./requirements.txt
RUN pip3 install -r requirements.txt


COPY rc.xml /etc/xdg/openbox/rc.xml
COPY menu.xml /etc/xdg/openbox/menu.xml

# Setup Nitrogen (Desktop background)
COPY wallpaper.png /tmp/wallpaper.png
COPY bg-saved.cfg /home/techuser/.config/nitrogen/bg-saved.cfg

# Copy Supervisor Configuration
COPY supervisord.conf /etc/supervisord.conf

# Allow any user to start the RDP server
# Depending on the base image used, Xwrapper.config may (not) be available and has to be created.
RUN sed -i 's/allowed_users=console/allowed_users=anybody/g' /etc/X11/Xwrapper.config \
    || echo "allowed_users=anybody" > /etc/X11/Xwrapper.config && \
    chmod 666 /etc/shadow

# Set permissions
RUN mkdir -p /run/xrdp/sockdir && \
    chown -R techuser /etc/xrdp /run/xrdp /var/log/xrdp* && \
    chown techuser /var/log

WORKDIR /home/techuser

COPY remote.sh .remote.sh
# COPY autostart /home/techuser/.config/openbox/autostart.sh
RUN chmod 755 .remote.sh /home/techuser/.config/openbox/autostart

COPY metrics.py .metrics.py
RUN chown techuser /home/techuser/.metrics.py

USER techuser
ENTRYPOINT [ "/home/techuser/.remote.sh" ]
