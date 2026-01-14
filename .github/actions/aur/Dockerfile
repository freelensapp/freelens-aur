FROM docker.io/library/archlinux:base-devel

RUN pacman -Syu --needed --noconfirm base-devel pacman-contrib namcap git sudo && \
  useradd -m builder && \
  echo 'builder ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers

WORKDIR /home/builder
USER builder

# Install yay
RUN git clone https://aur.archlinux.org/yay-bin.git
WORKDIR /home/builder/yay-bin
RUN makepkg -si --noconfirm

COPY entrypoint.sh /entrypoint.sh

ENTRYPOINT [ "/entrypoint.sh" ]
