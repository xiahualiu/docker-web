FROM node:20-slim

WORKDIR /src

# https://mediasoup.org/documentation/v3/mediasoup/installation/
ENV MEDIASOUP_SKIP_WORKER_PREBUILT_DOWNLOAD="true"

# Install necessary system packages and dependencies
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        python3 \
        python3-pip \
        ffmpeg \
    && rm -rf /var/lib/apt/lists/*

COPY ./mirotalksfu/package.json .
RUN npm install

# Cleanup unnecessary packages and files
RUN apt-get purge -y --auto-remove build-essential python3-pip \
&& npm cache clean --force \
&& rm -rf /tmp/* /var/tmp/* /usr/share/doc/*

# Copy the application code
COPY ./mirotalksfu/app app
COPY ./mirotalksfu/public public

# Set default command to start the application
CMD ["npm", "start"]
