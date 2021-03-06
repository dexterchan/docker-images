FROM ubuntu:xenial-20180228@sha256:e348fbbea0e0a0e73ab0370de151e7800684445c509d46195aef73e090a49bd6


USER root

# Install all OS dependencies for notebook server that starts but lacks all
# features (e.g., download as all possible file formats)
ENV DEBIAN_FRONTEND noninteractive
RUN apt-get update && apt-get -yq dist-upgrade \
 && apt-get install -yq --no-install-recommends \
    wget \
    bzip2 \
    ca-certificates \
    sudo \
    locales \
    fonts-liberation \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*

RUN echo "en_US.UTF-8 UTF-8" > /etc/locale.gen && \
    locale-gen

# Install Tini
RUN wget --quiet https://github.com/krallin/tini/releases/download/v0.10.0/tini && \
    echo "1361527f39190a7338a0b434bd8c88ff7233ce7b9a4876f3315c22fce7eca1b0 *tini" | sha256sum -c - && \
    mv tini /usr/local/bin/tini && \
    chmod +x /usr/local/bin/tini

# Configure environment
ENV CONDA_DIR=/opt/conda \
    SHELL=/bin/bash \
    NB_USER=pyconda \
    NB_UID=1000 \
    NB_GID=100 \
    LC_ALL=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    LANGUAGE=en_US.UTF-8
ENV PATH=$CONDA_DIR/bin:$PATH \
    HOME=/home/$NB_USER

ADD fix-permissions /usr/local/bin/fix-permissions
# Create pyconda user with UID=1000 and in the 'users' group
# and make sure these dirs are writable by the `users` group.
RUN useradd -m -s /bin/bash -N -u $NB_UID $NB_USER && \
    mkdir -p $CONDA_DIR && \
    chown $NB_USER:$NB_GID $CONDA_DIR && \
    chmod g+w /etc/passwd /etc/group && \
    fix-permissions $HOME && \
    fix-permissions $CONDA_DIR

USER $NB_UID

# Setup work directory for backward-compatibility
RUN mkdir /home/$NB_USER/work && \
    fix-permissions /home/$NB_USER

# Install conda as pyconda and check the md5 sum provided on the download site
ENV MINICONDA_VERSION 4.4.10
RUN cd /tmp && \
    wget --quiet https://repo.continuum.io/miniconda/Miniconda3-${MINICONDA_VERSION}-Linux-x86_64.sh && \
    echo "bec6203dbb2f53011e974e9bf4d46e93 *Miniconda3-${MINICONDA_VERSION}-Linux-x86_64.sh" | md5sum -c - && \
    /bin/bash Miniconda3-${MINICONDA_VERSION}-Linux-x86_64.sh -f -b -p $CONDA_DIR && \
    rm Miniconda3-${MINICONDA_VERSION}-Linux-x86_64.sh && \
    $CONDA_DIR/bin/conda config --system --prepend channels conda-forge && \
    $CONDA_DIR/bin/conda config --system --set auto_update_conda false && \
    $CONDA_DIR/bin/conda config --system --set show_channel_urls true && \
    $CONDA_DIR/bin/conda update --all --quiet --yes && \
    conda clean -tipsy && \
    rm -rf /home/$NB_USER/.cache/yarn && \
    fix-permissions $CONDA_DIR && \
    fix-permissions /home/$NB_USER

USER root

ENV DEBIAN_FRONTEND noninteractive

ENV ORACLE_INSTANTCLIENT_MAJOR 11.2
ENV ORACLE_INSTANTCLIENT_VERSION 11.2.0.4.0
ENV ORACLE /usr/local/oracle
ENV ORACLE_HOME $ORACLE/lib/oracle/$ORACLE_INSTANTCLIENT_MAJOR/client64
ENV LD_LIBRARY_PATH $LD_LIBRARY_PATH:$ORACLE_HOME/lib
ENV C_INCLUDE_PATH $C_INCLUDE_PATH:$ORACLE/include/oracle/$ORACLE_INSTANTCLIENT_MAJOR/client64

RUN apt-get update && apt-get install -y libaio1 \
            curl rpm2cpio cpio \
        && mkdir $ORACLE && TMP_DIR="$(mktemp -d)" && cd "$TMP_DIR" \
        && curl -L https://github.com/sergeymakinen/docker-oracle-instant-client/raw/assets/oracle-instantclient$ORACLE_INSTANTCLIENT_MAJOR-basic-$ORACLE_INSTANTCLIENT_VERSION-1.x86_64.rpm -o basic.rpm \
        && rpm2cpio basic.rpm | cpio -i -d -v && cp -r usr/* $ORACLE && rm -rf ./* \
        && ln -s libclntsh.so.11.1 $ORACLE/lib/oracle/$ORACLE_INSTANTCLIENT_MAJOR/client64/lib/libclntsh.so.$ORACLE_INSTANTCLIENT_MAJOR \
        && ln -s libocci.so.11.1 $ORACLE/lib/oracle/$ORACLE_INSTANTCLIENT_MAJOR/client64/lib/libocci.so.$ORACLE_INSTANTCLIENT_MAJOR \
        && curl -L https://github.com/sergeymakinen/docker-oracle-instant-client/raw/assets/oracle-instantclient$ORACLE_INSTANTCLIENT_MAJOR-devel-$ORACLE_INSTANTCLIENT_VERSION-1.x86_64.rpm -o devel.rpm \
        && rpm2cpio devel.rpm | cpio -i -d -v && cp -r usr/* $ORACLE && rm -rf "$TMP_DIR" \
        && echo "$ORACLE_HOME/lib" > /etc/ld.so.conf.d/oracle.conf && chmod o+r /etc/ld.so.conf.d/oracle.conf && ldconfig \
        && rm -rf /var/lib/apt/lists/* && apt-get purge -y --auto-remove curl rpm2cpio cpio

RUN apt-get update && apt-get install -yq --no-install-recommends graphviz \
        && apt-get clean && rm -rf /var/lib/apt/lists/*

USER $NB_USER

# Install Tensorflow
RUN pip install influxdb
RUN pip install pyreadline
RUN conda install --quiet --yes \
        'numpy' \
        'cx_Oracle=6.0*' \
        'matplotlib' \
        'pandas' \
        'scipy' \
        'seaborn' \
        'scikit-learn' \
        'scikit-image' \
        'tensorflow' \
        'keras' \
        'plotly' \
        'quandl' \
        'tweepy' \
        'textblob' \
        'PyYAML' && \
        conda clean -tipsy && \
        fix-permissions $CONDA_DIR

USER root

WORKDIR $HOME

# Configure container startup
ENTRYPOINT ["tini", "--"]
CMD ["start.sh"]

# Add local files as late as possible to avoid cache busting
COPY start.sh /usr/local/bin/
COPY start-notebook.sh /usr/local/bin/
COPY start-singleuser.sh /usr/local/bin/
#RUN fix-permissions /etc/jupyter/

# Switch back to pyconda to avoid accidental container runs as root
USER $NB_UID
