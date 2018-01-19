# osrm-express-server-demo

A brief example to combine OSRM with Express (NodeJs) to build your own
production-ready instance.

It is based on [Open Source Routing Machine (OSRM)](https://project-osrm.org)
and [Express](http://expressjs.com/).


## Requirements

In order to run this application you need to install _Docker_:

* Mac OSX via homebrew: `brew cask install docker`
* others: [https://docs.docker.com/engine/installation/](https://docs.docker.com/engine/installation/)

If the installation is successful, you should be able to run the following command
and get a similar response:

    docker version

    # output:
    Client:
     Version:      17.07.0-ce-rc1
     API version:  1.31
     Go version:   go1.8.3
     Git commit:   8c4be39
     Built:        Wed Jul 26 05:20:09 2017
     OS/Arch:      darwin/amd64
    Server:
     Version:      17.07.0-ce-rc1
     API version:  1.31 (minimum version 1.12)
     Go version:   go1.8.3
     Git commit:   8c4be39
     Built:        Wed Jul 26 05:25:01 2017
     OS/Arch:      linux/amd64
     Experimental: true


## Setup - Building the containers

* the main application: `docker build -t osrm-express-server -f Dockerfile.nodejs .`
* osmctools for data manipulation: `docker build -t osmium-tool -f Dockerfile.osmium-tool .`
* osrm-backend to generate the routable graph: `docker pull osrm/osrm-backend:v5.14.2`

## Data preprocessing

In order to run this application a routable graph generated by OSRM is required.
OSRM itself can build a graph from various sources, in the following OpenStreetMap (OSM)
data is used.

### Get OSM data for your region of interest

Download the region you are interested in (e.g. Europe) from [Geofabrik](http://download.geofabrik.de/) or similar
sources and save the result in the folder `data/`

    curl --create-dirs --output data/osrm/example-areas/europe-latest.osm.pbf http://download.geofabrik.de/europe-latest.osm.pbf


### Optional: Clip the OSM data

If your preferred region is not available, you can take a larger dataset and clip it to specific areas.
The following will use the `europe-latest.osm.pbf` dataset, extract Berlin and London and save it as a new
dataset.

We can now make use of the previously containerised `osmctools` to extract specific areas with a `.poly` file.
Read more about Poly-files [here](http://wiki.openstreetmap.org/wiki/Osmosis/Polygon_Filter_File_Format):

    docker run -it -v $(pwd)/data:/data osmium-tool extract -p /data/polyfiles/example-areas.poly /data/osrm/example-areas/europe-latest.osm.pbf -o /data/osrm/example-areas/example-areas.osm.pbf


### Create a routable graph

Once we have our dataset, we can pass it to the OSRM container to create a routable graph with a given profile (car, foot, bike, ...)
with the contraction hierachies algorithm:

    # extract the network and create the osrm file(s)
    docker run -it -v $(pwd)/data:/data osrm/osrm-backend:v5.14.2 osrm-extract -p /opt/car.lua /data/osrm/example-areas/example-areas.osm.pbf

    # create the graph
    docker run -it -v $(pwd)/data:/data osrm/osrm-backend:v5.14.2 osrm-contract /data/osrm/example-areas/example-areas.osrm

The dataset is now available at `./data/osrm/example-areas` and can be used by any OSRM instance running the same version.

## Starting the server

In order to launch the application with the specified dataset (see above), run
the container with the required environment variables, port mapping and volume bindings:

    docker run --env-file .env -d -p 5000:5000 -v $(pwd)/data:/data osrm-express-server

The `.env` file is a (modified) copy of the `.env.example` file contained in this repository.
Please adapt it according to your needs.

Also please make sure to map the correct volumes (host and container) as well as the ports in
order to access the server.

After the application has loaded the graph in memory, you can make requests to the server running at [http://localhost:5000](http://localhost:5000)

## API Endpoints

This example application has 3 different endpoints

* `GET /health`: A simple ping endpoint to check that the server is running.

* `POST /route`: Implements calls to `osrm.route` to calculate the way from A to B.

  _Example body_:
  ```
    { coordinates: [[13.3905, 52.5205], [13.3906, 52.5206]] }
  ```

* `POST /table`: Implements calls to `osrm.table` to get a travel times matrix for all provided locations.

  _Example body_:
  ```
    { coordinates: [[13.3905, 52.5205], [13.3906, 52.5206]] }
  ```

The source code should be simple enough to provide an overview and to get started for
extensions / own development.

## Testing

This repository also includes a small and simple test suite.
In order to run the tests in the container, execute the following commands:

    docker run -it --env-file .env -v $(pwd)/data:/data -p 5000:5000 osrm-express-server yarn lint
    docker run -it --env-file .env -v $(pwd)/data:/data -p 5000:5000 osrm-express-server yarn test

The test graph is a small subsample of areas in Berlin and London and is included in this repository.
Whenever you upgrade to a newer OSRM version, you need to rebuild the test graph as well to successfully run the tests again:

    docker run -it -v $(pwd)/data:/data osrm/osrm-backend:v5.14.2 osrm-extract -p /opt/car.lua /data/osrm/test/test.osm.pbf
    docker run -it -v $(pwd)/data:/data osrm/osrm-backend:v5.14.2 osrm-contract /data/osrm/test/test.osrm