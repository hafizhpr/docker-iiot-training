# Custom Node-RED image with the InfluxDB palette node pre-installed.
#
# Used by docs/12-building-custom-images.md. The default docker-compose.yml
# still uses the stock image, so students install the node by hand first and
# feel the problem this file solves.
FROM nodered/node-red:4.1.0-22

# The base image sets WORKDIR to /usr/src/node-red, where Node-RED's own
# package.json lives. Installing here bakes the node into the image itself,
# not into the /data volume.
RUN npm install --no-audit --no-fund --no-update-notifier \
    node-red-contrib-influxdb
