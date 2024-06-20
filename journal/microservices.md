# Microservices development

## Microservice Endpoints

There are two microservice endpoints:

1. The flights management microservice: _flyreserve-ms-flights_

2. The reservations management microservice: _flyreserve-ms-reservations_

We can describe the interactions represented by various jobs, using UML sequence diagrams in PlantUML format:

![seeds-interactions](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/uedwinc/flyreserve/main/seeds-interactions.puml)

The reservations microservice will be implemented in _Python_ and _Flask_, while the flights microservice will be implemented in _Node/Express.js_.

## The Database

We ensure the two microservices do not share any data space, so we implement them using entirely different backend data systems: Redis for the reservations and MySQL for flights.

1. Redis for the Reservations Data Model

2. MySQL Data Model for the Flights Microservice

## Code for the Microservices

### 1. Flights microservice

Create a new GitHub repository: flyreserve-ms-flights

Clone the following [repository](https://github.com/implementing-microservices/ms-flights) and change the remote origin url to the new GitHub repository

Final repository for the flights microservice: https://github.com/uedwinc/flyreserve-ms-flights

- Use GitPod to test the containers locally

- When everything is working, you should be able to access your /flights endpoint locally at a URL like the following: http://0.0.0.0:5501/flights?flight_no=AA34&departure_date_time=2020-05-17T13:20

![flight-no](../images/flight-no.png)

- The seat_maps endpoint should appear in a URL: http://0.0.0.0:5501/flights/AA2532/seat_map

![seat-map](../images/seat-map.png)

**Health checks**

To manage the life cycle of the containers that the app will be deployed into, most container-management solutions (e.g., Kubernetes) need a service to expose a health endpoint. In the case of Kubernetes, you should generally provide liveness and readiness endpoints.

A simple “Am I live?” check at **/ping** (known as a _liveness probe_ in Kubernetes) and a more advanced “Is the database also ready? Can I actually do useful things?” check (known as the _readiness probe_ in Kubernetes) at **/health** is implemented. Using two probes for overall health is very convenient since a microservice being up doesn’t always mean that it is fully functional. If its dependency, such as a database, is not up yet or is down, it won’t be actually ready for useful work.

If everything was done correctly and the microservice is up and running in a healthy way, if you now run `curl http://0.0.0.0:5501/health`, you should get a health endpoint output json.

If you run `curl http://0.0.0.0:5501/ping` instead, you should get a simpler output:

```json
{ "status": "pass" }
```

- We now have a fully functioning ms-flights microservice implemented with Node.js and MySQL

### 2. Reservations microservice

Implementing the flyreserve ms-reservations microservice in Python and Flask using the Redis data store

Create a new GitHub repository: flyreserve-ms-reservations

Clone the following [repository](https://github.com/implementing-microservices/ms-reservations) and change the remote origin url to the new GitHub repository

Final repository for the flights microservice: https://github.com/uedwinc/flyreserve-ms-reservations

You should be able to run `make` from the top level of the source code, which will build and run the project at `0.0.0.0:7701`.

If you encounter issues at any point or would like to check out the application logs for some reason, you can do this using the logs-app make target:

```sh
make logs-app
```

You may notice that the logs say the service is running on port 5000, but that is true inside the Docker container; it’s not port 5000 on the host machine! We map the standard Flask port 5000 to 7701 on the host machine (your machine). You can view the combined app and database logs by running `make logs`, or just the database logs by running `make logs-db`

- Now let’s run several curl commands to insert a couple of reservations:

```sh
curl --header "Content-Type: application/json" \
  --request PUT \
  --data '{"seat_num":"12B","flight_id":"werty", "customer_id": "dfgh"}' \
  # If you are using Gitpod, replace the next line with your Gitpod port url like https://7701-uedwinc-flyreservemsres-jku61f7g1e6.ws-eu114.gitpod.io/reservations
  http://0.0.0.0:7701/reservations
```

```sh
curl --header "Content-Type: application/json" \
  --request PUT \
  --data '{"seat_num":"12C","flight_id":"werty", "customer_id": "jkfl"}' \
  http://0.0.0.0:7701/reservations
```

![reserve-success](../images/reserve-success.png)

- We can also test that our protection against accidental double-bookings works. Let’s verify this by attempting to reserve an already reserved seat (e.g., 12C):

```sh
curl -v --header "Content-Type: application/json" \
  --request PUT \
  --data '{"seat_num":"12C","flight_id":"werty", "customer_id": "another"}' \
  http://0.0.0.0:7701/reservations
```

This will respond with HTTP 403 and an error message:

![cant-double-book](../images/cant-double-book.png)

To test the reservation retrieval endpoint, we can issue a curl command and verify that we receive the expected JSON response or confirm the endpoint on browser:

```sh
curl -v http://0.0.0.0:7701/reservations?flight_id=werty
```

![response-json](../images/response-json.png)

Now what we need to do is figure out a way to execute these two microservices (and any additional future components you may create) as a single unit. For this, we will introduce the notion of an “umbrella project”

## Hooking Services Up with an Umbrella Project

We need an easy-to-use umbrella project, one that can launch all of our microservice specific subprojects in one simple command and make them all work together nicely, until such time as we decided to shut down the umbrella project with all of its components.

To deploy an easy-to-use umbrella project, we’ll use the microservices workspace template available at https://github.com/inadarei/microservices-workspace and start a workspace for us at https://github.com/implementing-microservices/microservices-workspace

To check out repositories of individual microservices under the umbrella repository, we use the open source project [Faux Git Submodules](https://github.com/inadarei/faux-git-submodules)

The idea is to make it easy to descend into a subfolder of your workspace repository containing a microservice and treat it as a fully functioning repository, which you can update, commit code in, and push to.

Create a new GitHub repository: flyreserve-microservice-workspace

Clone the following [repository](https://github.com/implementing-microservices/ms-reservations) and change the remote origin url to the new GitHub repository

Final repository for the flights microservice: https://github.com/uedwinc/flyreserve-microservice-workspace

We’ll start by indicating the two microservice repos as the components of the new workspace, by editing the _fgs.json_ file to look something like the following:

```json
{
  "ms-flights" : {
    "url" : "https://github.com/uedwinc/flyreserve-ms-flights"
  },
  "ms-reservations" : {
    "url" : "https://github.com/uedwinc/flyreserve-ms-reservations"
  }
}
```

In the last configuration we indicated ms-flights and ms-reservations using the read-only “http://” protocol. This was done so that you can follow the example. In real projects, you would want to pull your repositories with the read/write “git://” protocol so you can modify them

Now that we have configured repos.json, let’s pull the ms-flights and ms-reservations microservices into the workspace:

```sh
make update
```

This operation also helpfully adds the checked-out repositories to the _.gitignore_ of the parent folder, to prevent the parent repository trying to double-commit them into the wrong place

We also need to edit the `bin/start.sh` and `bin/stop.sh` scripts to make changes from the default.

To keep things simple yet powerfully automated, our workspace setup is using the [Traefik edge router](https://traefik.io/traefik/) for seamless routing to the microservices. It gets installed by our [docker-compose.yml](https://github.com/inadarei/microservices-workspace/blob/master/docker-compose.yml) file

We will need to add Traefik-related labels to the `docker-compose.yml` files of both microservices to ensure proper routing of those services:

  1. _flyreserve-ms-flights/docker-compose.yaml_

```yml
services:
  ms-flights:
    container_name: ms-flights
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.ms-flights.rule=PathPrefix(`/flights`)"
```

  2. _flyreserve-ms-reservations/docker-compose.yaml_

```yml
services:
  ms-reservations:
    container_name: ms-reservations
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.ms-reservations.rule=PathPrefix(`/reservations`)"
```

We also need to update the umbrella project’s name (which serves as the namespace and network name for all services) in the workspace’s makefile, so that it says: `project:=flyreserve`.

Once you bring up the workspace by running `make start` at the workspace level, you will be able to access both microservices in their attached-to-workspace form

We mounted Traefik to local port 9080, making http://0.0.0.0:9080/ our base URI. Therefore, the following two commands are querying the reservations and flights systems:

```sh
curl http://0.0.0.0:9080/reservations?flight_id=qwerty
```

```sh
curl \
http://0.0.0.0:9080/flights?\
flight_no=AA34&departure_date_time=2020-05-17T13:20
```