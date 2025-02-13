########################################################################
# The base image derived from Debian buster slim
########################################################################

FROM ubuntu:focal as conda-base

LABEL maintainer="Jose Gomez-dans <j.gomez-dans@ucl.ac.uk"
ARG NB_USER="jovyan"
ARG NB_UID="1000"
ARG NB_GID="100"
ARG UHOME=/home/$NB_USER

# Fix DL4006
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

USER root

# Install all OS dependencies for notebook server that starts but lacks all
# features (e.g., download as all possible file formats)
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update \
 && apt-get install -yq --no-install-recommends \
    git \
    wget \
    bzip2 \
    ca-certificates \
    sudo \
    locales \
    fonts-liberation \
    run-one \
 && apt-get clean && rm -rf /var/lib/apt/lists/*
 
 ## Latex deps. Apparently, you need all of this!
 # texlive-xetex texlive-fonts-recommended texlive-generic-recommended

RUN echo "en_US.UTF-8 UTF-8" > /etc/locale.gen && \
    locale-gen

# Configure environment
ENV CONDA_DIR=/opt/conda \
    SHELL=/bin/bash \
    NB_USER=$NB_USER \
    NB_UID=$NB_UID \
    NB_GID=$NB_GID \
    LC_ALL=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    LANGUAGE=en_US.UTF-8
ENV PATH=$CONDA_DIR/bin:$PATH \
    HOME=$UHOME

# Copy a script that we will use to correct permissions after running certain commands
COPY Docker/fix-permissions /usr/local/bin/fix-permissions
RUN chmod a+rx /usr/local/bin/fix-permissions

# Enable prompt color in the skeleton .bashrc before creating the default NB_USER
RUN sed -i 's/^#force_color_prompt=yes/force_color_prompt=yes/' /etc/skel/.bashrc

# Create NB_USER wtih name jovyan user with UID=1000 and in the 'users' group
# and make sure these dirs are writable by the `users` group.
RUN echo "auth requisite pam_deny.so" >> /etc/pam.d/su && \
    sed -i.bak -e 's/^%admin/#%admin/' /etc/sudoers && \
    sed -i.bak -e 's/^%sudo/#%sudo/' /etc/sudoers && \
    useradd -m -s /bin/bash -N -u $NB_UID $NB_USER && \
    mkdir -p $CONDA_DIR && \
    chown $NB_USER:$NB_GID $CONDA_DIR && \
    chmod g+w /etc/passwd && \
    fix-permissions $HOME && \
    fix-permissions $CONDA_DIR

# as root
RUN mkdir -p /etc/jupyter/

USER $NB_UID
WORKDIR $HOME
ARG PYTHON_VERSION=default

# Setup work directory for backward-compatibility
RUN mkdir -p $UHOME/work && \
    fix-permissions $UHOME


# Install conda as jovyan and check the md5 sum provided on the download site
ENV MINICONDA_VERSION=4.8.3 \
    MINICONDA_MD5=751786b92c00b1aeae3f017b781018df \
    CONDA_VERSION=4.8.3

WORKDIR /tmp
RUN wget --quiet https://repo.continuum.io/miniconda/Miniconda3-py37_${MINICONDA_VERSION}-Linux-x86_64.sh && \
    echo "${MINICONDA_MD5} *Miniconda3-py37_${MINICONDA_VERSION}-Linux-x86_64.sh" | md5sum -c - && \
    /bin/bash Miniconda3-py37_${MINICONDA_VERSION}-Linux-x86_64.sh -f -b -p $CONDA_DIR && \
    rm Miniconda3-py37_${MINICONDA_VERSION}-Linux-x86_64.sh 
    
RUN echo "conda ${CONDA_VERSION}" >> $CONDA_DIR/conda-meta/pinned && \
    conda config --system --prepend channels conda-forge && \
    conda config --system --set auto_update_conda false && \
    conda config --system --set show_channel_urls true && \
    conda config --system --set channel_priority strict && \
    if [ ! $PYTHON_VERSION = 'default' ]; then conda install --yes python=$PYTHON_VERSION; fi && \
    conda list python | grep '^python ' | tr -s ' ' | cut -d '.' -f 1,2 | sed 's/$/.*/' >> $CONDA_DIR/conda-meta/pinned && \
    conda install --quiet --yes conda && \
    conda install --quiet --yes pip && \
    conda update --all --quiet --yes && \
    conda clean --all -f -y && \
    rm -rf $UHOME/.cache/yarn && \
    fix-permissions $CONDA_DIR && \
    fix-permissions $UHOME

# Install Tini
RUN conda install --quiet --yes 'tini=0.18.0' && \
    conda list tini | grep tini | tr -s ' ' | cut -d ' ' -f 1,2 >> $CONDA_DIR/conda-meta/pinned && \
    conda clean --all -f -y && \
    fix-permissions $CONDA_DIR && \
    fix-permissions $UHOME

# Install Jupyter Notebook, Lab, and Hub
# Generate a notebook server config
# Cleanup temporary files
# Correct permissions
# Do all this in a single RUN command to avoid duplicating all of the
# files across image layers when the permissions change

# Install Jupyter Notebook, Lab, and Hub
# Generate a notebook server config
# Cleanup temporary files
# Correct permissions
# Do all this in a single RUN command to avoid duplicating all of the
# files across image layers when the permissions change
RUN conda install --quiet --yes --freeze-installed \
    nomkl \
    'notebook=6.0.3' \
    'jupyterhub=1.1.0' \
    'jupyterlab=2.2.0'  && \
    conda clean --all -f -y && \
    npm cache clean --force --yes && \
    jupyter notebook --generate-config -y && \
    rm -rf $CONDA_DIR/share/jupyter/lab/staging && \
    rm -rf $UHOME/.cache/yarn && \
    fix-permissions $CONDA_DIR && \
    fix-permissions $UHOME

EXPOSE 8888

# Configure container startup
ENTRYPOINT ["tini", "-g", "--"]
CMD ["start-notebook.sh"]

# Copy local files as late as possible to avoid cache busting
COPY Docker/start.sh Docker/start-notebook.sh Docker/start-singleuser.sh /usr/local/bin/
COPY Docker/jupyter_notebook_config.py /etc/jupyter/

# Fix permissions on /etc/jupyter as root
USER root
RUN fix-permissions /etc/jupyter/

# Switch back to jovyan to avoid accidental container runs as root
USER $NB_UID

WORKDIR $HOME

########################################################################
# The course-specific docker file
########################################################################

FROM conda-base as uclgeog


LABEL maintainer="Jose Gomez-dans <j.gomez-dans@ucl.ac.uk"
ARG NB_USER="jovyan"
ARG NB_UID="1000"
ARG NB_GID="100"

# Fix DL4006
SHELL ["/bin/bash", "-o", "pipefail", "-c"]


# name your environment
ARG conda_env=uclgeog
ENV PATH=$CONDA_DIR/bin:$PATH \
    HOME=$HOME

RUN rm -rf "${HOME}"/tmp/ && mkdir -p "${HOME}"/tmp/

# Switch back to jovyan to avoid accidental container runs as root
USER $NB_UID

WORKDIR "${HOME}"


COPY Docker/environment.yml "${HOME}"/tmp/
RUN cd "${HOME}"/tmp/ && \
     conda env create -p $CONDA_DIR/envs/$conda_env -f environment.yml && \
     conda clean --all -f -y \
    && rm -rf "${HOME}"/tmp/environment.yml \
    && find /opt/conda/ -follow -type f -name '*.a' -delete \
    && find /opt/conda/ -follow -type f -name '*.pyc' -delete \
    && find /opt/conda/ -follow -type f -name '*.js.map' -delete



# create Python 3.x environment and link it to jupyter
RUN $CONDA_DIR/envs/${conda_env}/bin/python -m ipykernel install --user --name=${conda_env} && \
    fix-permissions $CONDA_DIR && \
    fix-permissions "${HOME}"

ENV PATH=$CONDA_DIR/envs/${conda_env}/bin:$PATH

ENV CONDA_DEFAULT_ENV=${conda_env}


RUN python -m pip install jupyterthemes && \
    python -m pip install --upgrade jupyterthemes && \
    python -m pip install jupyter_contrib_nbextensions && \
    jupyter contrib nbextension install --user && \
    rm -rf ${HOME}/.cache/pip
# enable the Nbextensions
RUN jupyter nbextension enable contrib_nbextensions_help_item/main && \
    jupyter nbextension enable autosavetime/main && \
    jupyter nbextension enable codefolding/main && \
    jupyter nbextension enable code_font_size/code_font_size && \
    jupyter nbextension enable code_prettify/code_prettify && \
    jupyter nbextension enable collapsible_headings/main && \
    jupyter nbextension enable comment-uncomment/main && \
    jupyter nbextension enable equation-numbering/main && \
    jupyter nbextension enable execute_time/ExecuteTime && \
    jupyter nbextension enable gist_it/main && \
    jupyter nbextension enable hide_input/main && \
    jupyter nbextension enable spellchecker/main && \
    jupyter nbextension enable toc2/main && \
    jupyter nbextension enable toggle_all_line_numbers/main && \
    jupyter nbextension enable exercise2/main  && \
    rm -rf "${HOME}"/.cache/pip
    


RUN mkdir -p "${HOME}"/notebooks
RUN fix-permissions "${HOME}"/notebooks/
COPY notebooks/* "${HOME}"/notebooks/
RUN jupyter trust notebooks/*ipynb 



WORKDIR $HOME
