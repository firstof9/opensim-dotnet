FROM mcr.microsoft.com/dotnet/runtime:8.0

LABEL maintainer="firstof9 <firstof9@gmail.com>" \
      version="0.9.3" \
      description="Robust Server 0.9.3"

ARG SOURCE=https://github.com/opensim/opensim/releases/latest/download/LastDotNetBuild.zip

# 1. Optimize apt and remove Mono keys (unnecessary for .NET 8)
RUN apt-get update && apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
    wget \
    unzip \
    ca-certificates \
    libgdiplus \
    && rm -rf /var/lib/apt/lists/*

# 2. Structured setup
RUN mkdir -p /home/opensim/opensim/bin/persistence

# 3. Streamlined download
RUN wget -O /tmp/opensim.zip "$SOURCE" && \
    unzip -d /home/opensim/opensim /tmp/opensim.zip && \
    rm /tmp/opensim.zip

# 4. Config injection
# Note: Ensure Robust.exe.config points to different log files than OpenSim
COPY Robust.exe.config /home/opensim/opensim/bin/

ENV DOTNET_SYSTEM_CONSOLE_ALLOW_ANSI_COLOR_REDIRECTION=false
ENV TERM=dumb

WORKDIR /home/opensim/opensim/bin

# 5. Direct execution (The Docker Way)
# This allows Docker to stop Robust gracefully without needing screen
CMD ["dotnet", "Robust.dll", "-console=basic"]
