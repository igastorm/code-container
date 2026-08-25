# ==========================================
# Version definition
# ==========================================
ARG GTEST_VERSION=1.18.0
ARG GTEST_SHA512=ba0f5769ccf34acf1bc72d1f7e9ffb8202176d02b64f6f3d9047accfc0cf9026ff5a653d24935e2705fff8709566676452616c93ca0ca6277f1e21d79b58a10a

ARG CODE_SERVER_VERSION=4.134.0
ARG CODE_SERVER_AMD64_SHA512=c361d9bc20053e891188713c0f099ff2b6c844fa35844eda6a85462bef6a9fcfd13b103d54765d2c809ac8570aebed37b44d20a268f275737ebe8aed797df55a
ARG CODE_SERVER_ARM64_SHA512=0a04d07e6fb972213d2c99b4243191ba898b9bdeb5d9f3a9dd76ea31f962dd997a84b3d6cbb8794de02fc1b5173ec25473be168c07b96cff27eb6d90cf230752

ARG CODELLDB_VERSION=1.12.3
ARG CODELLDB_X64_SHA512=1825e59f02dd57cb2c40e26dc238c65f5cdedeb53f71e5009c1f751070f2f304ab42c9f0af761c51fa4012132f66f90c2dd43181fca1de1660c18eb33d75b41b
ARG CODELLDB_ARM64_SHA512=355ddeb9a2abca95b72fa95ba685029f86c1fd4b6a4e834a11d8089a262269d3c535583d4f4b393f022094e43f81ea1eddca6f0164be2227815f47b9671a10df

# ==========================================
# Runtime
# ==========================================
FROM debian:trixie-slim@sha256:3a39a0592364683e6bab97937b72cad5a8fa6dcbbee90edb3bb48c7f8e94f258

ENV LANG=C.UTF-8
ENV LC_ALL=C.UTF-8

ARG GTEST_VERSION
ARG GTEST_SHA512

RUN echo 'Acquire::Languages "none";' > /etc/apt/apt.conf.d/99-no-languages && \
  apt-get update && apt-get install -y --no-install-recommends \
  tini sudo ca-certificates \
  libnss-extrausers \
  llvm \
  clang clang-tools clang-format clang-tidy clangd \
  libclang-rt-dev \
  lldb \
  lld \
  libc++-dev libc++abi-dev \
  ninja-build cmake curl python3 python3-pip python3-venv git ssh-client && \
  \
  # ==========================================
  # LLVM Native Toolchain
  # ==========================================
  CLANG_VER=$(clang --version | grep -oE 'clang version [0-9]+' | grep -oE '[0-9]+' | head -n1) && \
  CLANG_BIN_DIR="/usr/lib/llvm-${CLANG_VER}/bin" && \
  TRIPLE=$(clang -print-target-triple) && \
  mkdir -p "${CLANG_BIN_DIR}" && \
  echo "-fuse-ld=lld" > "${CLANG_BIN_DIR}/clang.cfg" && \
  echo "-rtlib=compiler-rt" >> "${CLANG_BIN_DIR}/clang.cfg" && \
  echo "-stdlib=libc++" > "${CLANG_BIN_DIR}/clang++.cfg" && \
  echo "-fuse-ld=lld" >> "${CLANG_BIN_DIR}/clang++.cfg" && \
  echo "-rtlib=compiler-rt" >> "${CLANG_BIN_DIR}/clang++.cfg" && \
  cd "${CLANG_BIN_DIR}" && \
  ln -sf clang.cfg clang-${CLANG_VER}.cfg && \
  ln -sf clang++.cfg clang++-${CLANG_VER}.cfg && \
  ln -sf clang.cfg ${TRIPLE}-clang.cfg && \
  ln -sf clang++.cfg ${TRIPLE}-clang++.cfg && \
  ln -sf clang.cfg ${TRIPLE}-clang-${CLANG_VER}.cfg && \
  ln -sf clang++.cfg ${TRIPLE}-clang++-${CLANG_VER}.cfg && \
  # ==========================================
  # GoogleTest (libc++ & Disabled exception)
  # ==========================================
  curl -fsSL https://github.com/google/googletest/archive/refs/tags/v${GTEST_VERSION}.tar.gz -o gtest.tar.gz && \
  echo "${GTEST_SHA512} gtest.tar.gz" | sha512sum -c - && \
  tar -xzf gtest.tar.gz && cd googletest-${GTEST_VERSION} && mkdir build && cd build && \
  cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DCMAKE_CXX_FLAGS="-stdlib=libc++ -fno-exceptions" \
  -DCMAKE_INSTALL_PREFIX=/usr/local \
  .. && \
  ninja install && \
  cd ../.. && \
  rm -rf googletest-${GTEST_VERSION} gtest.tar.gz && \
  # ==========================================
  (getent group messagebus || groupadd -r messagebus) && \
  (getent passwd messagebus || useradd -r -g messagebus -s /usr/sbin/nologin messagebus) && \
  apt-get clean && \
  find /usr/share/locale -mindepth 1 -maxdepth 1 -exec rm -rf {} + && \
  rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* /usr/share/doc/* /usr/share/man/*

RUN sed -i 's/passwd:.*files/& extrausers/' /etc/nsswitch.conf && \
  sed -i 's/group:.*files/& extrausers/' /etc/nsswitch.conf && \
  sed -i 's/shadow:.*files/& extrausers/' /etc/nsswitch.conf && \
  echo "%sudo ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/sudo_nopasswd && \
  chmod 0440 /etc/sudoers.d/sudo_nopasswd && \
  sed -i '1i auth sufficient pam_extrausers.so nullok' /etc/pam.d/common-auth && \
  sed -i '1i account sufficient pam_extrausers.so' /etc/pam.d/common-account && \
  rm -rf /var/log && ln -s /tmp/log /var/log && \
  rm -rf /run && ln -s /tmp/run /run && \
  rm -rf /var/lib/extrausers && ln -s /tmp/extrausers /var/lib/extrausers

HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD ["sh", "-c", "curl -fsS http://localhost:3000 -o /dev/null || exit 1"]

COPY entrypoint.sh /usr/local/bin/entrypoint.sh
COPY settings.json /usr/local/etc/settings.json
RUN chmod +x /usr/local/bin/entrypoint.sh
# ==========================================
# code-server
# ==========================================
ARG CODE_SERVER_VERSION
ARG CODE_SERVER_AMD64_SHA512
ARG CODE_SERVER_ARM64_SHA512

ARG CODELLDB_VERSION
ARG CODELLDB_X64_SHA512
ARG CODELLDB_ARM64_SHA512

RUN ARCH=$(uname -m) && \
  if [ "$ARCH" = "x86_64" ]; then \
  CS_ARCH="amd64"; \
  CS_SHA512="${CODE_SERVER_AMD64_SHA512}"; \
  CODELLDB_ARCH="x64"; \
  CODELLDB_SHA512="${CODELLDB_X64_SHA512}"; \
  elif [ "$ARCH" = "aarch64" ]; then \
  CS_ARCH="arm64"; \
  CS_SHA512="${CODE_SERVER_ARM64_SHA512}"; \
  CODELLDB_ARCH="arm64"; \
  CODELLDB_SHA512="${CODELLDB_ARM64_SHA512}"; \
  else \
  echo "Unsupported architecture: $ARCH" && exit 1; \
  fi && \
  curl -fsSL "https://github.com/coder/code-server/releases/download/v${CODE_SERVER_VERSION}/code-server-${CODE_SERVER_VERSION}-linux-${CS_ARCH}.tar.gz" -o /tmp/code-server.tar.gz && \
  echo "${CS_SHA512} /tmp/code-server.tar.gz" | sha512sum -c - && \
  tar -xzf /tmp/code-server.tar.gz -C /usr/local && \
  mv /usr/local/code-server-${CODE_SERVER_VERSION}-linux-${CS_ARCH} /usr/local/code-server && \
  curl -fsSL "https://github.com/vadimcn/codelldb/releases/download/v${CODELLDB_VERSION}/codelldb-linux-${CODELLDB_ARCH}.vsix" -o /tmp/codelldb.vsix && \
  echo "${CODELLDB_SHA512} /tmp/codelldb.vsix" | sha512sum -c - && \
  mkdir -p /tmp/vscode-exts && \
  /usr/local/code-server/bin/code-server \
  --extensions-dir /tmp/vscode-exts \
  --install-extension llvm-vs-code-extensions.vscode-clangd \
  --install-extension ms-vscode.cmake-tools \
  --install-extension ms-vscode.hexeditor \
  --install-extension /tmp/codelldb.vsix \
  --install-extension ms-python.python \
  --install-extension ms-python.debugpy \
  --install-extension charliermarsh.ruff && \
  mv /tmp/vscode-exts/* /usr/local/code-server/lib/vscode/extensions/ && \
  rm -rf /tmp/code-server.tar.gz /tmp/codelldb.vsix /tmp/vscode-exts

EXPOSE 3000
ENTRYPOINT ["/usr/bin/tini", "--", "/usr/local/bin/entrypoint.sh"]
