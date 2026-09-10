# JSON-RPC server for the Hiero SDK TCK.
#
# Build context is the repository root, so build it from there:
#   docker build -f src/tck/Dockerfile -t hiero-sdk-cpp-tck .
#
# x86_64 only: the build uses the linux-x64-release preset (x64-linux vcpkg triplet).

FROM ubuntu:24.04 AS build

# 4-core runners OOM when ninja fans out over the generated protobuf translation
# units, so the job count is capped rather than left at ninja's default of N+2.
ARG BUILD_JOBS=3
ARG CMAKE_PRESET=linux-x64-release

ENV DEBIAN_FRONTEND=noninteractive

# perl builds vcpkg's openssl; zip and linux-libc-dev are both asserted by
# SystemLibraries.cmake; autoconf/automake/libtool build secp256k1 and apr.
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    ca-certificates \
    cmake \
    ninja-build \
    pkg-config \
    git \
    curl \
    zip \
    unzip \
    tar \
    autoconf \
    automake \
    libtool \
    linux-libc-dev \
    perl \
    python3 \
    && rm -rf /var/lib/apt/lists/*

RUN cmake --version && ninja --version && g++ --version && git --version

WORKDIR /hiero-sdk-cpp

COPY . .

# vcpkg is a submodule. It is already populated when the context was checked out
# with submodules; otherwise it is fetched here, so a plain checkout also builds.
RUN if [ ! -f vcpkg/bootstrap-vcpkg.sh ]; then \
      git submodule update --init --recursive; \
    fi

RUN ./vcpkg/bootstrap-vcpkg.sh -disableMetrics

# BUILD_TCK is off by default; without it the server target is never generated.
RUN cmake --preset "${CMAKE_PRESET}" -DBUILD_TCK=ON

# Ninja Multi-Config: the configuration comes from the build preset, and must be
# repeated on the install or CMake looks for Debug artifacts that were not built.
RUN cmake --build --preset "${CMAKE_PRESET}" -j "${BUILD_JOBS}"
RUN cmake --install "build/${CMAKE_PRESET}" --config Release --prefix /install --strip

# Fails the build here rather than at container start if the layout moved, and
# records the runtime libraries the final stage has to provide.
RUN test -x /install/tck/hiero-sdk-cpp-tck && ldd /install/tck/hiero-sdk-cpp-tck


FROM ubuntu:24.04

RUN apt-get update && apt-get install -y --no-install-recommends \
    libstdc++6 \
    libatomic1 \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY --from=build /install/tck/ /app/tck/

EXPOSE 8544

# The server takes its port as argv[1], not from the environment. The wrapper
# maps TCK_PORT onto it so the container is configured the same way as the
# other SDK TCK images.
ENTRYPOINT ["/bin/sh", "-c", "exec /app/tck/hiero-sdk-cpp-tck \"${TCK_PORT:-8544}\""]
