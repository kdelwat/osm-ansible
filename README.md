# OpenStreetMap Ansible playbook

This playbook is useful for setting up a complete, self-contained instance of OpenStreetMap, including:

- the read/write API
- the web interface
- a tile rendering pipeline
- PostgreSQL for data storage

Installing the stack manually takes a LOT of time, so hopefully this speeds up the process somewhat.

## Structure

The playbook assumes a single machine is used for all components. It could probably be tweaked to move heavier components to their own machine if needed.

A quick overview:

- The Apache web server sits in front of `renderd` and Mapnik. When tile requests hit the server, the `mod_tile` extension forwards them to `renderd`, which generates the tiles and returns them to the user. Generated tiles are cached on disk.

- The data for the tiles is contained in a PostGIS/PostgreSQL database (the `gis` table), and can be imported from OSM Planet files (see below)

- The stylesheet is built using [openstreetmap-carto](https://github.com/gravitystorm/openstreetmap-carto) and is used when creating the tile images

- Apache also serves the OpenStreetMap web application, which handles editing the map data.

_Important_: The web application and render pipeline actually use independent copies of map data (in incompatible formats). See below for how to persist edits from the application to the tile generator.

## Usage

### Requirements

You will need a server running Ubuntu 18.04. I used a VPS with 4GB of RAM and 2 vCPUs, which seems to work OK, although the more powerful the machine the faster tile generation will run.

It's probably a good idea to mount an external volume to store the PostgreSQL database. After importing OSM data for Oceania, my database was around ~40GB.

You will also need:

- Ansible install locally
- Python installed on the VPS (`apt-get install python`)

### Running the playbook

1. Create a file called `hosts` in this directory, containing your server's IP address like:

   ```
   [osm]
   78.46.xxx.xxx
   ```

2. Run the playbook: `ansible-playbook -i hosts site.yml`

### Importing initial data

If you want to prepopulate the server with OSM data, you will need a Planet file in PBF format.

You can download the full Planet (47GB) [here](https://planet.openstreetmap.org/) or a regional extract [here](http://download.openstreetmap.fr/extracts/). The more heavily-mapped the area, the larger the extract will be.

Initially, import the Planet into the OSM application database (command from [here](https://gis.stackexchange.com/a/169943)).

```
osmosis --read-pbf <PLANET_FILE>.osm.pbf \
  --write-apidb host="localhost" database="openstreetmap" \
  user="root" password="osm12345" validateSchemaVersion="no"
```

If you've changed any of the defaults in `defaults/main.yml`, you will need to change them in the command above.

### Updating tile data

After making edits to the OSM data, you will want to reflect that in the generated tiles. To convert OSM data to PostGIS:

```
# Export updated planet file
osmosis --read-apidb database="openstreetmap" user=root password="osm12345" validateSchemaVersion="no" \
        --write-pbf file="/mapserver/planet/planet.osm.pbf"

# Convert to PostGIS and load into database
osm2pgsql --create --database gis --slim --hstore --multi-geometry \
          --tag-transform-script /mapserver/styles/standard/openstreetmap-carto.lua \
          --style /mapserver/styles/standard/openstreetmap-carto.style "/mapserver/planet/planet.osm.pbf"

# Clear cache
rm -rf /mapserver/website/tiles/standard/*
```

(commands from [here](https://wiki.openstreetmap.org/wiki/User:Ika-chan!/Fantasy_maps_with_OSM_software#References))

### Improving performance

You can decrease tile rendering time by making some tweaks to PostgresSQL. See [here](https://ircama.github.io/osm-carto-tutorials/tile-server-ubuntu/).

# Thanks

Thanks to user Ika-chan! from the OpenGeoFiction community, who documented most of the setup and usage process [here](https://wiki.openstreetmap.org/wiki/User:Ika-chan!/Fantasy_maps_with_OSM_software#References). This playbook really just automates their hard work and brings it up to date with changes since the tutorial was written.
