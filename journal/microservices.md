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

**Health checks**

(...)

### 2. Reservations microservice

Implementing the flyreserve ms-reservations microservice in Python and Flask using the Redis data store

Create a new GitHub repository: flyreserve-ms-reservations

Clone the following [repository](https://github.com/implementing-microservices/ms-reservations) and change the remote origin url to the new GitHub repository

Final repository for the flights microservice: https://github.com/uedwinc/flyreserve-ms-reservations

## Hooking Services Up with an Umbrella Project