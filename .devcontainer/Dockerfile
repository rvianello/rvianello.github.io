ARG fedora_release=31
ARG rdkit_release=2020.03.3
FROM rvianello/fedora-rdkit-python:${fedora_release}-${rdkit_release}

# This Dockerfile image has a non-root user with sudo access. Use the "remoteUser"
# property in devcontainer.json to use it. On Linux, the container user's GID/UIDs
# will be updated to match your local UID/GID (when using the dockerFile property).
# See https://aka.ms/vscode-remote/containers/non-root-user for details.
ARG USERNAME=vscode
ARG USER_UID=1000
ARG USER_GID=$USER_UID

# Configure and install packages
RUN dnf install -y \
    git 

# Create a non-root user to use if preferred - see https://aka.ms/vscode-remote/containers/non-root-user.
RUN groupadd --gid $USER_GID $USERNAME \
    && useradd -s /bin/bash --uid $USER_UID --gid $USER_GID -m $USERNAME \
    # [Optional] Add sudo support for the non-root user
    && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME\
    && chmod 0440 /etc/sudoers.d/$USERNAME

COPY requirements.txt .

RUN python3 -m venv --system-site-packages /nikola \
    && . /nikola/bin/activate \
    && pip install -r ./requirements.txt
