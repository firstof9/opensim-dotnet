FROM mcr.microsoft.com/dotnet/runtime:8.0

# Using Labels instead of Maintainer
LABEL maintainer="firstof9 <firstof9@gmail.com>" \
      version="0.9.3" \
      description="Opensim 0.9.3 release"

ARG SOURCE=https://github.com/opensim/opensim/releases/latest/download/LastDotNetBuild.zip

# Combine apt commands to keep the image slim
RUN apt-get update && apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
    wget \
    unzip \
    ca-certificates \
    libgdiplus \
    && rm -rf /var/lib/apt/lists/*

# Setup directory structure
RUN mkdir -p /home/opensim/opensim/bin/persistence

# Download and Unzip in one layer to avoid keeping the zip in image history
RUN wget -O /tmp/opensim.zip "$SOURCE" && \
    unzip -d /home/opensim/opensim /tmp/opensim.zip && \
    rm /tmp/opensim.zip

# Copy your custom configs
COPY OpenSim.exe.config /home/opensim/opensim/bin/

# Disable ANSI color codes to stop "junk" data in docker logs
ENV DOTNET_SYSTEM_CONSOLE_ALLOW_ANSI_COLOR_REDIRECTION=false
ENV TERM=xterm

# We no longer need the .sh script if we run dotnet directly
WORKDIR /home/opensim/opensim/bin

# Use EXEC form for CMD to pass signals (SIGTERM) correctly to dotnet
CMD ["dotnet", "OpenSim.dll", "-console=basic"]
