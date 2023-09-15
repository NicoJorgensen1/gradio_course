##########################################################################################################################################################################################
# BUILD THIS IMAGE WITH ARGUMENTS:    docker build -f .devcontainer/Dockerfile -t gradio_course_image .
# RUN THIS IMAGE WITH ARGUMENTS:      docker run -it --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 --network host -p 6006:6006 \
#                                             --gpus all --privileged -v $IMAGE_DATASETS:/Image_Datasets gradio_course_image bash \
#                                             --ARG1 VALUE1 --ARG2 VALUE2

##########################################################################################################################################################################################
########################################################### First choose all the arguments used to build this docker container ###########################################################

# At first we use the pytorch prebuilt image from NVIDIA 
ARG FROM_IMAGE=nvcr.io/nvidia/pytorch:23.04-py3

# The Python version to install
ARG PYTHON_VERSION=3.10

# Select the username, userID and groupID for the user which will be created inside the container
ARG USERNAME=nico
ARG USER_UID=1000
ARG USER_GID=$USER_UID

# The working directory at which we will start inside the container 
ARG WORKING_DIR=/home/$USERNAME/gradio_course




##########################################################################################################################################################################################
######################################## Create the first layer of the image  => Here all ubuntu packages and python packages should be installed ########################################
FROM $FROM_IMAGE as ubuntu_cacher

# Assure the timezone is european => CPH 
ENV TZ=Europe/Copenhagen 
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone && date 

# Install ubuntu packages 
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get -y update \
    && apt-get -y --no-install-recommends install software-properties-common \
    && add-apt-repository -y ppa:deadsnakes/ppa \
    && apt-get -y update \
    && apt-get -y upgrade \
    && apt-get -y --no-install-recommends install \
        build-essential \
        ca-certificates \
        cmake \
        cmake-data \
        pkg-config \
        libcurl4 \
        libsm6 \
        libxext6 \
        libssl-dev \
        libffi-dev \
        libxml2-dev \
        libxslt1-dev \
        zlib1g-dev \
        unzip \
        curl \
        git \
        wget \
        sudo \
        ffmpeg \
        usbutils \
        graphviz
RUN sudo apt update -y && sudo apt install ssh -y




##########################################################################################################################################################################################
##################################### Create the second layer of the image => here the required python version will be installed and set as default ######################################
FROM ubuntu_cacher as python_installer
ARG PYTHON_VERSION

# Install the specified python version
RUN sudo apt-get install -y \
    python${PYTHON_VERSION} \
    python${PYTHON_VERSION}-dev \
    python${PYTHON_VERSION}-distutils 
RUN sudo ln -s /usr/bin/python${PYTHON_VERSION} /usr/local/bin/python \
    && ln -s /usr/local/lib/python${PYTHON_VERSION} /usr/local/lib/python \
    && rm /usr/bin/python3 \
    && ln -s /usr/bin/python3.9 /usr/bin/python3 \
    && curl https://bootstrap.pypa.io/get-pip.py | python \
    && rm -rf /var/lib/apt/lists/*

ENV DEBIAN_FRONTEND "" 




##########################################################################################################################################################################################
######################################################### Create the third layer of the image => here a new user will be created #########################################################
FROM python_installer as user_cacher
ENV USER=$USERNAME

ARG USERNAME
ARG USER_UID
ARG USER_GID

# Natively the image runs as root. That is not best practise and will give some problems. Thus we need to create and add another user (that we here also make root).
RUN groupadd --gid $USER_GID $USERNAME \
    && useradd --uid $USER_UID --gid $USER_GID -m $USERNAME \
    && usermod -aG root $USERNAME \
    # [Optional] Add sudo support. Omit if you don't need to install software after connecting.
    && apt-get update -y \
    && apt-get install -y sudo \
    && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME
RUN usermod -s /bin/bash $USERNAME

# Add the user to the plugdev group, allowing for the user to access the USB ports 
RUN usermod -aG plugdev $USERNAME

# Install the bash-auto complete 
RUN sudo apt-get install bash-completion && sudo apt-get install --reinstall bash-completion




##########################################################################################################################################################################################
####################################################### Create the fourth layer of the image  => Here the container is ready to run ######################################################
FROM user_cacher as standard_repo_cacher
ARG PYTHON_VERSION
ARG BITBUCKET_REPO_URL

# As the docker build process can't get access to the ssh keys of the host computer, an ssh-copy wasn't feasible.
# Instead now an app password associated with the 'nico_j_capra' account, which provides read-only access for repositories-only, is used.
# In this way the standard_functions repo is installed as a python package 

COPY .devcontainer/install_csf.sh install_csf.sh

# Now we run a shell script to install the standard_function package in order to not exploit the environmental variables used inside 
RUN chmod +x install_csf.sh \
    && /bin/bash install_csf.sh \
    && rm install_csf.sh




##########################################################################################################################################################################################
################################################# Create the fifth layer of the image  => Here the python requirements will be installed #################################################
FROM standard_repo_cacher as python_cacher
ARG USERNAME
ARG PYTHON_VERSION
ARG WORKING_DIR

# Set the workdir and copy everything into the correct place (remember the .dockerignore for any larger, unwanted files)
WORKDIR $WORKING_DIR

# Install the requirements 
COPY .devcontainer/requirements_gradio.txt requirements_gradio.txt 

# Install python dependencies
RUN python${PYTHON_VERSION} -m pip install --upgrade pip
RUN python${PYTHON_VERSION} -m pip install -r requirements_gradio.txt




##########################################################################################################################################################################################
####################################################### Create the sixth layer of the image  => Here the container is ready to run #######################################################
FROM python_cacher as runner 
ARG WORKING_DIR
ARG USERNAME

# Set the working directory and make the user own the entire directory 
ENV USER=${USERNAME}
ENV IMAGE_DATASETS=/Image_Datasets

# Copy all necessary items into the WORKING_DIR and provide file ownership to the user 
WORKDIR $WORKING_DIR
COPY . $WORKING_DIR
RUN cd && sudo chmod a+rwx .
