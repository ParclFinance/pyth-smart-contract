ARG SOLANA_VERSION
ARG SOLANA_DOCKER_IMAGE_HASH
FROM solanalabs/solana:v${SOLANA_VERSION}@sha256:${SOLANA_DOCKER_IMAGE_HASH}

# Redeclare SOLANA_VERSION in the new build stage.
# Persist in env for docker run & inspect.
ARG SOLANA_VERSION
ENV SOLANA_VERSION="${SOLANA_VERSION}"

RUN apt-get update
RUN apt-get install -qq \
  cmake \
  curl \
  g++ \
  gcc-multilib \
  git \
  libzstd1 \
  libzstd-dev \
  python3-pytest \
  python3-pytest-asyncio \
  python3-websockets \
  sudo \
  zlib1g \
  zlib1g-dev \
  qtbase5-dev \
  qtchooser \
  qt5-qmake \
  qtbase5-dev-tools \
  libqt5websockets5-dev\
  libclang-dev\
  python3-pip 

# Install jcon-cpp library
RUN git clone https://github.com/joncol/jcon-cpp.git /jcon-cpp && cd /jcon-cpp && git checkout 2235654e39c7af505d7158bf996e47e37a23d6e3 && mkdir build && cd build && cmake .. && make -j4 && make install

# Grant sudo access to pyth user
RUN echo "pyth ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
RUN useradd -m pyth

# Fixes a bug in the solana docker image
# https://github.com/solana-labs/solana/issues/20577
RUN mkdir /usr/bin/sdk/bpf/dependencies \
    && chmod -R 777 /usr/bin/sdk/bpf


USER pyth
WORKDIR /home/pyth

RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \
  | sh -s -- -y 


RUN echo "\n\
export PATH=\"\${PATH}:\${HOME}/pyth-client/build:\${HOME}/.cargo/bin\"\n\
export PYTHONPATH=\"\${PYTHONPATH:+\$PYTHONPATH:}\${HOME}/pyth-client\"\n\
" >> .profile

COPY --chown=pyth:pyth . pyth-client/

# Build off-chain binaries.
RUN cd pyth-client && ./scripts/build.sh

RUN ./pyth-client/scripts/get-bpf-tools.sh
# Apply patch to solana
RUN ./pyth-client/scripts/patch-solana.sh

# Build and test the oracle program.
RUN cd pyth-client && ./scripts/build-bpf.sh .
RUN cd pyth-client && ./scripts/check-size.sh
# Run aggregation logic tests
RUN cd pyth-client && ./scripts/run-aggregation-tests.sh
RUN /bin/bash -l -c "pytest-3 --pyargs pyth"

ENTRYPOINT []
CMD []
