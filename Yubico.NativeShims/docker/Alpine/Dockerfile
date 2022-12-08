# These arguments are passed in from the console or by Docker-CLI itself. User/Group ID
# is used to run the shell as the host's user identity. Artifact_dir should be set to
# the NuGet runtime-id for this platform (e.g. alpine-x64) to aid in artifact discovery
# in the GitHub Action workflow.
# We use Bionic as the basis for our image. Bionic is the last Alpine long-term support
# release that is both still supported and still supports i386 processors. Once Bionic
# leaves support, we can consider upgrading to the next LTS release.
ARG YUBICO_ARTIFACT_DIR=$YUBICO_ARTIFACT_DIR
ARG YUBICO_USER_ID=$YUBICO_USER_ID
ARG YUBICO_GROUP_ID=$YUBICO_GROUP_ID
FROM alpine:latest AS env

# Set up basic environment variables such as the path.
ENV PATH=/usr/local/bin:$PATH

# Add the host user and group to the image.
RUN addgroup --GID ${YUBICO_GROUP_ID:-1000} -S local && adduser --UID ${YUBICO_USER_ID:-1000} --SHELL /bin/ash -H -S ${YUBICO_GROUP_ID:-1000} -G local
# original: RUN groupadd -f -g ${YUBICO_GROUP_ID:-1000} local && useradd -o -u ${YUBICO_USER_ID:-1000} -g ${YUBICO_GROUP_ID:-1000} -s /bin/sh local

# Install build tools
RUN apk update && \
    apk --no-cache add \
    wget \
    ca-certificates \
    gnupg \
    tesseract-ocr \
    alpine-sdk \
    ninja && \
    apk upgrade

# Install latest CMake
RUN apk update && \
    apk --no-cache add \
    cmake && \
    apk upgrade

# Install build dependencies
# This is where we should add any additional dependencies needed by Yubico.NativeShims.
# We could use vcpkg to help with dependencies, but for Linux, the distro's package
# manager is still almost always going to be the easiest way of finding the necessary
# headers and pre-built libraries. Be sure to use the -dev packages, as these typically
# denote the package that contains headers and libs.
RUN apk update && \
    apk --no-cache add \
    libpcsclite-dev \
    libssl-dev \
    apk upgrade

# Snapshot the base environment. If we ever decide to cache our images in a container
# registry, `env` is the target we'd want to capture. The dependencies will be installed
# but we have not yet copied the source code to build into the image. That happens in
# this (devel) stage.
FROM env AS devel
# Let's work out of a folder that's out of the way on the filesystem.
WORKDIR /home/build
# Copy the host context (source code) into the image. See the notes in the shell script
# that invokes Docker to see the other end of specifying the context. Copies all of the
# host context (recursively) into the current working dir in the Docker image.
COPY . .
RUN rm -rf artifacts

# Build the Yubico.NativeShims shared object
# Now we take the `devel` target, and fork another image for building. This way, we can
# quickly roll back a failed build and retry (or try interactively). Put all of the
# build instructions in this stage. For now, this simply means generating the CMake
# cache, and building using CMake. We move the build artifacts into a well known
ARG YUBICO_ARTIFACT_DIR=$YUBICO_ARTIFACT_DIR
# location to help the artifact stage.
FROM devel AS build
RUN cmake -S . -B build_out -DCMAKE_BUILD_TYPE=Release
RUN cmake --build build_out --target all -v
RUN mkdir -p /home/build/artifacts/${YUBICO_ARTIFACT_DIR:-alpine-x64} \
    && cp /home/build/build_out/*.so /home/build/artifacts/${YUBICO_ARTIFACT_DIR:-alpine-x64}

# Copy over the build artifacts to a blank image. This way we can easily retrieve the
# build results without copying all of the previous image's filesystem. `Scratch` is
# a completely blank image. We then use the `COPY` instruction to pull only the files
# we care about into this blank space. The `--output` argument to the `docker` command
# specifies what we do with this result.
ARG YUBICO_ARTIFACT_DIR=$YUBICO_ARTIFACT_DIR
FROM scratch AS build_install
COPY --from=build /home/build/artifacts/${YUBICO_ARTIFACT_DIR:-alpine-x64}/ .