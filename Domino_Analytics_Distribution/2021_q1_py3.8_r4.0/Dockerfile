

FROM ubuntu:20.04
LABEL maintainer="Domino Data Lab <support@dominodatalab.com>"
LABEL "com.dominodatalab.www"="Domino Data Lab"
LABEL description="Domino Analytics Distribution"

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

#######################################################
#              Utilities required by Domino           #
#######################################################
ARG DEBIAN_FRONTEND=noninteractive
ENV DEBIAN_FRONTEND=noninteractive
# Create the Ubuntu User
RUN \
  groupadd -g 12574 ubuntu && \
  useradd -u 12574 -g 12574 -m -N -s /bin/bash ubuntu && \
# UPDATE, UPGRADE, and ADD repositories
  apt-get update -y && \
  apt-get -y install software-properties-common && \
  apt-get -y upgrade && \
# CONFIGURE locales
  apt-get install -y locales && \
  locale-gen en_US.UTF-8 && \
  dpkg-reconfigure locales && \
# INSTALL common things
  apt-get -y install sudo build-essential wget curl apt-utils net-tools libzmq3-dev ed git ca-certificates iputils-ping dnsutils netcat pwgen apt-transport-https vim python3-pip jq unzip krb5-user zsh fonts-powerline htop && \
  apt-get -y --no-install-recommends install libssl-dev libxml2-dev libxt-dev libssh2-1-dev libcurl4-openssl-dev libsasl2-dev && \
# AWS CLI
  curl -sL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" &&   unzip awscliv2.zip && \
  aws/install  && \
  rm -rf awscliv2.zip aws && \
  echo 'export PYTHONIOENCODING=utf-8' >> /home/ubuntu/.domino-defaults && \
  echo 'export LANG=en_US.UTF-8' >> /home/ubuntu/.domino-defaults && \
  echo 'export JOBLIB_TEMP_FOLDER=/tmp' >> /home/ubuntu/.domino-defaults && \
  echo 'export LC_ALL=en_US.UTF-8' >> /home/ubuntu/.domino-defaults && \
  locale-gen en_US.UTF-8
ENV LANG en_US.UTF-8

###############################################################
#                    sudo bug only 20.04                      #
###############################################################
RUN echo "Set disable_coredump false" >> /etc/sudo.conf

RUN echo "ubuntu    ALL=NOPASSWD: ALL" >> /etc/sudoers

###############################################################
#                 Python 3.8 and Miniconda                    #
###############################################################
# Inspiration: https://github.com/jupyter/docker-stacks/blob/master/base-notebook/Dockerfile
# https://repo.continuum.io/miniconda/
ENV CONDA_DIR /opt/conda
ENV PATH $CONDA_DIR/bin:$PATH
ENV MINICONDA_VERSION latest
ENV MINICONDA_MD5 1314b90489f154602fd794accfc90446111514a5a72fe1f71ab83e07de9504a7
# Set environment variables so they are available in Domino runs and workspaces
RUN \
    echo 'export CONDA_DIR=/opt/conda' >> /home/ubuntu/.domino-defaults && \
    echo 'export PATH=$CONDA_DIR/bin:$PATH' >> /home/ubuntu/.domino-defaults && \
    echo 'export PATH=/home/ubuntu/.local/bin:$PATH' >> /home/ubuntu/.domino-defaults && \
# Install Python and Mini-conda
    cd /tmp && \
    wget --quiet https://repo.anaconda.com/miniconda/Miniconda3-${MINICONDA_VERSION}-Linux-x86_64.sh && \
    echo "${MINICONDA_MD5} *Miniconda3-${MINICONDA_VERSION}-Linux-x86_64.sh" | sha256sum -c - && \
    /bin/bash Miniconda3-${MINICONDA_VERSION}-Linux-x86_64.sh -f -b -p $CONDA_DIR && \
    rm Miniconda3-${MINICONDA_VERSION}-Linux-x86_64.sh && \
# Make all conda packages up-to-date. pip upgrade will NOT work as it will break things in a massive way
    conda update --update-all && \
# Specify the python version
    conda install python && \
# Make conda folder permissioned for ubuntu user
    chown ubuntu:ubuntu -R $CONDA_DIR && \
# Use Mini-conda's pip
    ln -s $CONDA_DIR/bin/pip /usr/bin/pip && \
    ln -s $CONDA_DIR/bin/python /usr/local/bin/python && \
# Use Mini-conda's python  
    ln -s $CONDA_DIR/anaconda/bin/python /usr/local/bin/python3  && \
# Set permissions
    chown -R ubuntu:ubuntu  $CONDA_DIR && \
# Upgrade Pip
    pip install --no-cache-dir --upgrade pip && \
#Cleanup
    conda clean --all --force-pkgs-dirs --yes && \
    rm -Rf /tmp/*

# Add various package dependencies and useful libraries
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        libhdf5-dev libyaml-dev pkg-config libfuse-dev cups libcups2-dev python-gi \
        python-gi-cairo python3-gi python3-gi-cairo gir1.2-gtk-3.0 libsmbclient-dev python-debian swig \
        curl libcurl4-openssl-dev libssl-dev pandoc libgmp-dev libmpfr-dev libmpc-dev && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* && \
    rm -Rf /tmp/*

# Install Domino Dependencies 
# Packages used for model APIs and Apps and some extras that are useful
COPY requirements.txt .
RUN $CONDA_DIR/bin/conda install -c conda-forge uWSGI && \
    pip install --no-cache-dir -r requirements.txt && \
    pip cache purge && \
    chown ubuntu:ubuntu -R $CONDA_DIR

# Install packages used in Domino quick-start project
RUN pip install --no-cache-dir git+https://github.com/dominodatalab/python-domino.git
# Configure matplotlib
RUN mkdir -p /home/ubuntu/.config/matplotlib && \
    echo "backend : Cairo" > /home/ubuntu/.config/matplotlib/matplotlibrc  && \
    sed -i 's/backend      : TkAgg/backend      : Cairo/g' /opt/conda/lib/python3.8/site-packages/matplotlib/mpl-data/matplotlibrc

#######################################################
#  Install drivers for common data source connections #
#######################################################
#SQL 
RUN \
    apt-get update -y && apt-get install tdsodbc -y && \
    apt-get install -y --no-install-recommends libodbc1 unixodbc freetds-common freetds-dev && \
    cd /tmp && \
    wget -q ftp://ftp.unixodbc.org/pub/unixODBC/unixODBC-2.3.9.tar.gz && \
    tar -xf unixODBC-2.3.9.tar.gz && \
    cd unixODBC-2.3.9 && \
    ./configure --enable-gui=no --enable-drivers=no --enable-iconv --with-iconv-char-enc=UTF8 --with-iconv-ucode-enc=UTF16LE --libdir=/usr/lib/x86_64-linux-gnu --prefix=/usr --sysconfdir=/etc --enable-stats=no && \
    make && \
    make install && \
    ln -s /lib/x86_64-linux-gnu/libssl.so.1.0.0 /lib/x86_64-linux-gnu/libssl.so.10 && \
    ln -s /lib/x86_64-linux-gnu/libcrypto.so.1.0.0 /lib/x86_64-linux-gnu/libcrypto.so.10 && \
    ldconfig && \
    rm -rf /tmp/* && \
#Cleanup
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* && \
    rm -Rf /tmp/*

# Oracle & PyOracle
RUN \
    mkdir -p /opt/oracle && \
    wget -q https://s3-us-west-2.amazonaws.com/domino-deployment/2016/02/22/instantclient-basic-linux.x64-12.1.0.2.0.zip -O /opt/oracle/instantclient-basic-linux.x64-12.1.0.2.0.zip && \
    wget -q https://s3-us-west-2.amazonaws.com/domino-deployment/2016/02/22/instantclient-sdk-linux.x64-12.1.0.2.0.zip -O /opt/oracle/instantclient-sdk-linux.x64-12.1.0.2.0.zip && \
    apt-get update -y && apt-get install libaio1 && \
    mkdir -p /opt/oracle/instantclient_12_1 && \
    cd /opt/oracle && \
    unzip instantclient-basic-linux.x64-12.1.0.2.0.zip && \
    unzip instantclient-sdk-linux.x64-12.1.0.2.0.zip && \
    cd /opt/oracle/instantclient_12_1 && \
    ln -s libclntsh.so.12.1 libclntsh.so && \
    ln -s libocci.so.12.1 libocci.so && \
    echo "export LD_LIBRARY_PATH=/opt/oracle/instantclient_12_1:\${LD_LIBRARY_PATH:-}" >> /home/ubuntu/.domino-defaults && \
    echo 'export OCI_LIB=/opt/oracle/instantclient_12_1' >> /home/ubuntu/.domino-defaults && \
    echo 'export OCI_INC=/opt/oracle/instantclient_12_1/sdk/include' >> /home/ubuntu/.domino-defaults && \
    echo "export PATH=/opt/oracle/instantclient_12_1:\${PATH:-}" >> /home/ubuntu/.domino-defaults && \
    chown -R ubuntu:ubuntu /opt/oracle/instantclient_12_1 && \
    echo '/opt/oracle/instantclient_12_1' > /etc/ld.so.conf.d/oracle-instantclient.conf && \
    ldconfig -v && \
    export PATH=/opt/oracle/instantclient_12_1:${PATH:-} && \
    export LD_LIBRARY_PATH=/opt/oracle/instantclient_12_1:${LD_LIBRARY_PATH:-} && \
    pip install --no-cache-dir cx_Oracle && \
    cd /home/ubuntu && \
    rm -rf /var/lib/apt/lists/* && \
    rm -rf /opt/oracle/instantclient-basic-linux.x64-12.1.0.2.0.zip && \
    rm -rf /opt/oracle/instantclient-sdk-linux.x64-12.1.0.2.0.zip
# Install PostgreSQL client
RUN apt-key adv --keyserver keyserver.ubuntu.com --recv-keys B97B0AFCAA1A47F044F244A07FCC7D46ACCC4CF8 && \
    echo "deb http://apt.postgresql.org/pub/repos/apt/ focal-pgdg main" > /etc/apt/sources.list.d/pgdg.list && \
    apt-get update && apt-get -y -q install postgresql-13 postgresql-client-13 postgresql-contrib-13 && \
    rm -rf /var/lib/apt/lists/*

#######################################################
#      Installing Notebooks,Workspaces,IDEs,etc       #
#######################################################
# Add workspace install and configuration scripts
RUN \
    cd /tmp && \
    git clone https://github.com/captnbp/workspace-configs.git /var/opt/workspaces && \
    rm -rf /var/opt/workspaces/workspace-logos && \
    # Jupyter
    chmod +x /var/opt/workspaces/jupyter/install && \
    /var/opt/workspaces/jupyter/install && \
# VScode
    pip install --no-cache-dir python-language-server autopep8 flake8 pylint && \
    pip install --no-cache-dir git+git://github.com/dominodatalab/jupyter_codeserver_proxy-.git && \
    chmod +x /var/opt/workspaces/vscode/install && \
    /var/opt/workspaces/vscode/install && \
    rm -Rf /opt/code-server/lib/vscode/extensions/npm && \
# Jupyterlab
    chmod +x /var/opt/workspaces/Jupyterlab/install && \
    /var/opt/workspaces/Jupyterlab/install && \
# Jupyter server proxy
    conda install -y -c conda-forge nodejs && \
    pip install --no-cache-dir jupyter-server-proxy && \
    jupyter labextension install @jupyterlab/server-proxy && \
    chown -R ubuntu:ubuntu /home/ubuntu/ ${CONDA_DIR} && \
#Cleanup
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* && \
    rm -Rf /tmp/*

#######################################################
#                     Some fixes                      #
#######################################################
RUN pip install --no-cache-dir pyodbc psycopg2-binary confluent-kafka pyarrow pymssql tensorflow

# Fix some npm modules manually that ship with ALL current versions of nodejs
RUN npm init -y && \
    npm --depth 9999 update && npm -g --depth 9999 update && \
# DOT-PROP. One can force replace it like this.
    rm -Rf /usr/lib/node_modules/npm/node_modules/dot-prop && \
    wget -qO- https://github.com/sindresorhus/dot-prop/archive/v6.0.1.tar.gz -P /usr/lib/node_modules/npm/node_modules/dot-prop | tar zxvf - --strip-components=1 && \
# Some manual clean-ups
# Let's start with VSCode Handlebars Extension
    rm -Rf /opt/code-server/lib/vscode/extensions/handlebars && \
# Fix handlebars which does not work to update with npm but needs yarn
    curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - && \
    echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list && \
    apt update && apt install -y yarn && \
    apt update && apt install -y --no-install-recommends yarn && \
    yarn add handlebars && \
    yarn upgrade --latest && \
    /opt/code-server/node_modules/yarn/bin/yarn && \
    /opt/code-server/node_modules/yarn/bin/yarn upgrade --latest && \
    yarn cache clean && \
# Fix http-proxy module
    sed -i 's/\"1.18.0"/\"1.18.1"/g' /opt/code-server/node_modules/http-proxy/package.json && \
    npm upgrade --prefix /opt/code-server/node_modules/http-proxy/ && \
# Fix serialize-javascript
    sed -i 's/\"2.1.2"/\"4.0.0"/g' /opt/conda/share/jupyter/lab/staging/node_modules/serialize-javascript/package.json && \
    npm upgrade --prefix /opt/conda/share/jupyter/lab/staging/node_modules/serialize-javascript/ && \
#Cleanup
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* && \
    rm -Rf /tmp/*
# Install Domino De

###############################################################
#                       Jupyterlab Git                        #
###############################################################
RUN pip install --no-cache-dir --upgrade jupyterlab-git && jupyter lab build

###############################################################
#                         Clean Up                            #
###############################################################
# Removing coursier due to security vulnerabilities
RUN rm -rf /home/ubuntu/.cache/coursier
# Remove execution permission from conda binary to prevent Anaconda license violations
#RUN sed -i '/export PATH\=\$CONDA_DIR\/bin\:\$PATH/d' /home/ubuntu/.domino-defaults
RUN chmod 755 $CONDA_DIR/bin/conda
