# switch2OSMinfo
Additional information to primarliy accompany [Overv/openstreetmap-tile-server](https://github.com/Overv/openstreetmap-tile-server), found on [Switch2OSM](https://switch2osm.org/serving-tiles/using-a-docker-container/). All credits due to thier hard work!

Current Tile Server Version: **v2.20, (Aug 8, 2022)**

Info current as of: **25/01/2023**

- if version has changed on the above github, please inform me on Discord or via Issues. I will check for any required updates.

I recommend thoroughly reading the above pages for the basis of this information. This summary is provided to help avoid several of the pitfalls I encountered by providing a fully prepared docker-compose file and step-by-step instructions.

During the database creation and initial tile rendering processes, you will use a LOT of resources, and exceeding overall server limits is quite easy. Server used for reference:

- 12 thread, 3.5GHz (6 cores)
- 32GB DDR4
- 2x512GB NVMe SSD in Raid 0

This machine was also running other services accounting for 1/4 to 1/2 of its available resources. If using a dedicated machine you could also adjust for more. I was importing North America and rendering much of it - for which I am linking a tool to save a lot of time I wasted. You can do smaller extracts with much less resources, so adjust as needed. 

I limited the RAM allotment during import to half the server total (`OSM2PGSQL_EXTRA_ARGS: -C 16384`) but likely could have went to 60 or 70%. During rendering I limited to 75% of threads and it kept my RAM and CPU near max for several days, while still running its usual services.

# Getting Started
```
mkdir OSMTileServer && cd OSMTileServer
git clone https://github.com/alx77/render_list_geo.pl.git
chmod +x render_list_geo.pl/render_list_geo.pl
```

Planet: Set PBF line to `https://planet.openstreetmap.org/pbf/planet-latest.osm.pbf` and delete the POLY line. Add the related extra options specified below the compose file.

Region: Visit [Geofabrik](https://download.geofabrik.de/) and determine the urls for your region's pbf & poly.

Copy the following contents using your favorite editor and update per your environment/region. Save in the OSMTileServer directory:

# docker-compose.yml
```
version: '3.5'
services:
  osmtileserver:
    image: overv/openstreetmap-tile-server
    container_name: osmtileserver
    restart: "no"
    volumes:
      - /home/<username>/OSMTileServer/render_list_geo.pl:/render_list_geo.pl
      - osm-data:/data/database/
      - osm-tiles:/data/tiles/
    shm_size: '4gb'
    ports:
      - "8080:80"
    environment:
      OSM2PGSQL_EXTRA_ARGS: -C 16384
      THREADS: 9
      DOWNLOAD_PBF: https://download.geofabrik.de/north-america-latest.osm.pbf
      DOWNLOAD_POLY: https://download.geofabrik.de/north-america.poly
      REPLICATION_URL: https://planet.openstreetmap.org/replication/minute/
      MAX_INTERVAL_SECONDS: 60
      UPDATES: enabled
      ALLOW_CORS: enabled
    command: "import"
volumes:
  osm-data:
  osm-tiles:
```

Extra Options:

If you are importing the whole planet, flat nodes is recommended. Add `FLAT_NODES: enabled` in the environment section and consider disabling PSQL's autovaccum with `AUTOVACUUM: off`

If you wish to connect to the postgres database (user: renderer), add `- "5432:5432"` to the ports section and consider changing the default password from renderer with `PGPASSWORD: secret`

There are more options but I didn't find them relevant to my objective. You can adapt them if needed from the github docker instructions in the intro links.

# Starting & Rendering

Start the import process and attach to the logs to observe it starting properly. Remove sudo if your user can run docker commands directly.

`sudo docker-compose up -d && sudo docker-compose logs -tf`

There will be several times where it seems like it is doing nothing. Many stages are hours long. You can check your server usage in another terminal but it should exit if there are any crashes. Sit back and wait for it to exit with successful table closing messages and code 0.

Once complete, edit your docker-compose.yml:

- `restart: unless-stopped`
- `command: "run"`
- any thread or memory adjustments for production

Start the tileserver with `sudo docker-compose up -d && sudo docker-compose logs -tf`. It should say it is ready after the database startup, and will begin doing the minutely update replication. You can cancel your log viewing once it is ready.

I found using the built in slippy-map unreliable for rendering but great for viewing afterwards. Thankfully the default server includes a perfect tool for this, render_list - which we have earlier cloned an extra utility to get the most of.

Connect directly to the running container `sudo docker exec -it osmtileserver bash`

Render the 'easy layers' in full. You CAN do more than layer 10 but they start to take a long time. `-n` refers to the number of threads to run, I matched it to the threads specified in compose. You may also specify `-l` with a number which represents a total server load limit.

`render_list -n 9 -a -f -t /data/tiles -z 0 -Z 10`

Use the 'Extra Tool' to render specific areas. I'm also crazy and rendered all the way to layer 20 for each city. You can set your deepest layer by changing the `-Z` option.

In a web browser, visit http://bboxfinder.com - draw a rectangle over the area you wish to render. Copy the string of coordinates beside 'Box', pasting after the `-x` and replacing the commas with `-y`, `-X`, and `-Y` in that order.

ex: `-89.762421,47.966939,-89.101868,48.576607` becomes

`-x -89.762421 -y 47.966939 -X -89.101868 -Y 48.576607`
```
cd render_list_geo.pl
./render_list_geo.pl -f -n 9 -t /data/tiles -z 11 -Z 20 -x <formatted paste>
```

Repeat as needed for as many areas/cities as you wish and have imported.

# Accessing the Tiles

Your tiles will now be available at http://localhost:8080/tile/{z}/{x}/{y}.png. The demo map in leaflet-demo.html will then be available on http://localhost:8080.

Use nginx/apache reverse proxy blocks to access externally/use HTTPS.

