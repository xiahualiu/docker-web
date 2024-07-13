FROM node:18-slim

WORKDIR /src

# https://mediasoup.org/documentation/v3/mediasoup/installation/
ENV MEDIASOUP_SKIP_WORKER_PREBUILT_DOWNLOAD="true"

COPY ./mirotalksfu/package.json .

RUN apt-get update && \
    apt-get install -y build-essential python3 python3-pip

RUN npm install --loglevel verbose && \
    npm cache clean --force && \
    rm -rf /tmp/* /var/lib/apt/lists/* /var/tmp/* /usr/share/doc/*

COPY ./mirotalksfu/app app
COPY ./mirotalksfu/public public

RUN ls

CMD npm start
