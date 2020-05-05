# switch2OSMinfo
Additional information to accompany https://switch2osm.org/serving-tiles/using-a-docker-container/ 

I recommend thoroughly reading the above page, and the linked pages found on it for the basis of this information. Thuis summary is provided to help avoid several of the pitfalls I encountered and to provide a fully prepared docker-compose file.

During the database creation and initial tile rendering processes, you will use a LOT of resources, and exceeding overall server limits is quite easy. The following information is based on a 12 thread, 3.5GHz server with 32GB DDR4 & 2x512GB NVMe SSD in Raid 0, importing North America and rendering much of it - for which I am linking a tool to save alot time I wasted. You can do smaller extracts with much less resources, so adjust as needed. This machine is also running RDM & various bots, so if using a dedicated machine you could also adjust for more.

I limited the RAM allotment during import to half the server total but likely could have went to 60 or 70%. In my final test I did not use flat nodes, but if you go for anything more than a continent, uncomment those lines/flags accordingly to make and keep the flat node file volume. I think you may have to remove the commented flags on the OSM2PSQL_EXTRA_ARGS lines, not 100% if it'll just ignore them as is as it is not in my final production file, added for example.

During rendering I limited to 75% of threads and it kept my RAM and CPU near max for several days (while still running the aforementioned)

This is prepared on an Ubuntu 18 server with a standard, sudo enabled user. Adjust paths as needed. In order to avoid file permission issues I let the compose create my volumes, trying to specify gave me issues.

`mkdir OSMTileServer && cd OSMTileServer`
`git clone https://github.com/alx77/render_list_geo.pl.git`

Follow the steps on the switch2osm readme to wget your region extract and poly file to the current directory from geofabrik. If doing the whole planet, you can find the pbf download at https://ftpmirror.your.org/pub/openstreetmap/pbf/
Remove the UPDATES flag and you will have to research how to update via osmosis or similar tools if doing whole planet. Can't say how much I don't recommend this though, unless needed and you have a monster server with several TB of storage.

Copy the following contents using your favorite editor

# docker-compose.yml
```
version: '3.5'
services:
  osmtileserver:
    image: overv/openstreetmap-tile-server
    container_name: osmtileserver
    restart: unless-stopped
    volumes:
      - /home/<username>/OSMTileServer/north-america-latest.osm.pbf:/data.osm.pbf
      - /home/<username>/OSMTileServer/north-america.poly:/data.poly
      - /home/<username>/OSMTileServer/render_list_geo.pl:/render_list_geo.pl
      - openstreetmap-data:/var/lib/postgresql/12/main
      - openstreetmap-rendered-tiles:/var/lib/mod_tile
#      - openstreetmap-nodes:/nodes
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    shm_size: '4gb'
    ports:
      - "127.0.0.1:8080:80"
      - "127.0.0.1:5432:5432"
    environment:
      PGPASSWORD: YourStrongPassw0rd!
      OSM2PGSQL_EXTRA_ARGS: -C 16384 # --flat-nodes /nodes/flat_nodes.bin
      THREADS: 9
      UPDATES: enabled
      ALLOW_CORS: enabled
    command: "run"
  osmimport:
    image: overv/openstreetmap-tile-server
    container_name: osmimport
    restart: "no"
    volumes:
      - /home/<username>/OSMTileServer/north-america-latest.osm.pbf:/data.osm.pbf
      - /home/<username>/OSMTileServer/north-america.poly:/data.poly
      - openstreetmap-data:/var/lib/postgresql/12/main
      - openstreetmap-rendered-tiles:/var/lib/mod_tile
#      - openstreetmap-nodes:/nodes
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    shm_size: '1gb'
    ports:
      - "127.0.0.1:5432:5432"
    environment:
      PGPASSWORD: YourStrongPassw0rd!
      OSM2PGSQL_EXTRA_ARGS: -C 16384 # --flat-nodes /nodes/flat_nodes.bin
      THREADS: 12
      UPDATES: enabled
    command: "import"
volumes:
  openstreetmap-data:
  openstreetmap-rendered-tiles:
#  openstreetmap-nodes:
```

# Starting & Rendering

Start the import process and attach to the logs to observe it starting properly. Due to some limitation of docker logs, it will not display the import detail progress until completed, or if you stop and start the docker-compose logs command it'll very awkwardly display the most current point if not using follow. Remove sudo if your user can run docker commands directly.

`sudo docker-compose up -d osmimport && sudo docker-compose logs -f`

Once it says 'Using PBF parser', if nothing has thrown any errors, you can use Ctrl+C to return to your prompt. Using `sudo docker-compose logs` will allow you to see the 'progress bar' if desired. Otherwise, just sit back and wait for it to exit with successful table closing messages and code 0.

Start the tileserver itself with `sudo docker-compose up -d osmtileserver && sudo docker-compose logs -f` to observe the rendering process. I found using the built in slippy-map unreliable, but thankfully the default server includes a perfect tool for this, render_list - which we have earlier cloned an extra utility to get the most of.

Open a second terminal/SSH and connect directly to the running container 
`sudo docker exec -it osmtileserver bash`
If you are only concerned with a single area or have used a small extract, skip this next step and go directly to using the extra tool

Render the 'easy layers' in full, run this command from anywhere in the container. You CAN do more than layer 10 but they start to take a long time. 'n' refers to the number of threads to run, I matched it to the threads specified in compose.
`render_list -n 9 -a -f -m ajt -z 0 -Z 10`

Use the 'Extra Tool' to render specific areas. 'n' refers to the number of threads to run, I matched it to the threads specified in compose. I'm also crazy and rendered all the way to layer 20 for each city :D
In a web browser, visit http://bboxfinder.com - draw a rectangle over the area you wish to render. Replace the numbers below with the numbers generated per the example
![Starting & Rendering](assets/example.png?raw=true)
`cd render_list_geo.pl`
`./render_list_geo.pl -n 9 -m ajt -z 11 -Z 20 -x <1> -X <2> -y <3> -Y <4>`

Repeat as needed for as many areas/cities as you wish and have imported.
