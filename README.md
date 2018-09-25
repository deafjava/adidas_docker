
# Adidas Coding Challenge

## 1. Instructions to run the microservices locally with `docker`

To run, your machine must have [__Java 8__](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html), [__docker__](https://docs.docker.com/install/) and [__git__](https://git-scm.com/downloads) installed

For docker, an OS Linux is recommended, due to its network better behaviour. __Ubuntu 18.04__ has been used. Although Mac is on a Unix System, but has its issues over docker network, so use the alternative way shown below to run by Mac.

### 1.1 Get the [_Routes_ Microservice](https://github.com/deafjava/adidas_route.git)

* Get the project: `git clone git@github.com:deafjava/adidas_route.git`
* Go to the project root: `cd adidas_route/`
* Build the project and generate docker image: `./gradlew build docker`

### 1.2 Get the [_Itineraries_ Microservice](https://github.com/deafjava/adidas_itineraries.git)

* Get the project: `git clone git@github.com:deafjava/adidas_itineraries.git`
* Go to the project root: `cd adidas_itineraries/`
* Build the project and generate docker image: `./gradlew build docker`


### 1.3 Get the [_URL Discovery_ Microservice](https://github.com/deafjava/adidas_eureka.git)

* Get the project: `git clone git@github.com:deafjava/adidas_eureka.git`
* Go to the project root: `cd adidas_eureka/`
* Build the project and generate docker image: `./gradlew build docker`

### 1.4 Get the [_docker-compose.yml_](https://github.com/deafjava/adidas_docker.git) file to run all above projects and bind them each others

* Get the project: `git clone git@github.com:deafjava/adidas_docker.git`
* Go to the project root: `cd adidas_docker/`
* Run with `docker-compose up -d`
* _Itineraries Service_ will be accessible on http://192.168.3.12:8082, _Routes Service_ will be accessible on http://192.168.3.11:8081 but secured - you may enter using "**admin**" as username and "**adidas!**" as password and _Discovery Service_ will be accessible on http://192.168.3.15:8088.

It shall take a while to make everything up - about 1:30 minutes. Don't forget to back to root after building each project - `cd ..`

Using a REST client software, such as [POSTMAN](https://www.getpostman.com/), or with a modern [Firefox](https://www.mozilla.org/en-US/firefox/new/), or elsewhat, then make a `GET http://192.168.3.12:8082/itinerary/bytime/MAD` to watch it.
The documentation of the Itineraries API Service lies on http://192.168.3.12:8082/swagger-ui.html

### 1.5 Alternative way to run the project

All steps above of getting the project remain the same, but instead of building the *docker* image, do:

* `./gradlew build` of each project
* Run with `java -jar build/libs/{generated_project_name}.jar &` - the ampersand will allow you to use the terminal, but it's recommended to run each of them in a separated tab, it's cleaner.

The steps above must be done by the root of each project.

All of project will be acessible by http://localhost/ instead, keeping the port and path.

## 2. Developing the Solution

By the Coding Challenge Description, I understood that it's a Flight Itinerary Consulting Service, that giving a origin city, using the IATA code, as this code behaves as a unique ID for City/Airport by convection, split into two main Microservices - one for providing detail of a route, other one for providing better readable with one way sorted by shortest time, other one way sorted by quantities of connections, both in a crescent way.

### 2.1 Database

I decided to go with *MySQL* flavor, due to its facilities to work in development, using locally with *MariaDB* docker container (a MySQL fork project), and to present the project to evaluating using in memory DBMS *H2* with *MySQL* mode to facilitate project running, acting like a *MySQL* locally installed.

I prepared a migration to pre-populate sample *route* data, using the Flyway technology, with DDL and DML SQL commands.

### 2.2 Microservices Development

All Microservices were made using **Spring Boot** with **Java 8**, using **Gradle** as build automation and dependency management, chose instead of *Maven*, because of better performance, flexibility, readability - see [Maven vs. Gradle](https://gradle.org/maven-vs-gradle/), beyond of **Docker** that has been asked for the project, this is a powerful tool to work with microservices. **docker-compose** has been used to assemble all microservices with rules and commands described by a *yml* file to make they run. Finally, a **Spring Cloud** was applied to make communications between microservices soft and effective by *Eureka* service, a project booted by *Netflix*, in a few words, to allow flexibility of URL addressing without microservices to know which URL to use, just to know the service name.

To make sorting solution, I used the Java 8 Stream, the code below is the heart of Itineraries Microservice:

```java
package com.adidas.travel.service;

import com.adidas.travel.domain.Itinerary;
import com.adidas.travel.domain.Route;
import lombok.AllArgsConstructor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.Comparator;
import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;

@Service
@AllArgsConstructor
public class ItineraryServiceImpl implements ItineraryService {

    @Autowired
    private OriginDestinyService originDestinyService;

    @Override
    public List<Itinerary> listSortedByTime(String iata) {

        List<Route> routes = originDestinyService.getRoutesByOrigin(iata);

        return routes
                .stream()
                .map(Itinerary::of)
                .sorted(Comparator.comparing(Itinerary::getDuration))
                .collect(Collectors.toList());
    }

    @Override
    public List<Itinerary> listSortedByConnections(String iata) {

        List<Route> routes = originDestinyService.getRoutesByOrigin(iata);

        return routes
                .stream()
                .map(Itinerary::of)
                .sorted((o1, o2) -> {
                    Optional<List<Itinerary>> connectionsOpt1 = Optional.ofNullable(o1.getConnections());
                    Optional<List<Itinerary>> connectionsOpt2 = Optional.ofNullable(o2.getConnections());
                    if (!connectionsOpt1.isPresent() && !connectionsOpt2.isPresent()) {
                        return 0;
                    }
                    if (!connectionsOpt1.isPresent()) {
                        return -1;
                    }
                    if (!connectionsOpt2.isPresent()) {
                        return 1;
                    }
                    int cs1 = connectionsOpt1.get().size();
                    int cs2 = connectionsOpt2.get().size();
                    return Integer.compare(cs1, cs2);
                })
                .collect(Collectors.toList());
    }
}
```

For shortest time, I used the built in `Comparator.comparing`, but for connections, had to make more detailed for error proof.

I used the Intellij IDE to develop the solution.

### 2.3 Architecture Principles
#### 2.3.1 Reusability
##### 2.3.1.1 Spring Data
I used the **Spring Data** to abstract the responsibility of database querying creation, and avoid boilerplate DAO development. When a project has its great need to make hard changes throughout different DBMS, with considerably different SQL syntaxes, we can go with *Liquibase* instead of *Flyway* to make the migration with less worry of kind of technology, the **Spring Data**, when used at least as maximum the *HQL*, without going with native SQL, the project will suffer less harm to make changes.

In the *Route Microservice*, I separated the *database* matter to a module, to allow a future addiction of a different perspective of database when a need may happen.

Finally, using __Spring__ itself is a great manner to reuse the code, with capabilities of *DI* and reusing its templates for various purposes.  
##### 2.3.1.2 TestNG
Due to its rich possibilities in comparison with the popular *JUnit*, this Test Framework has better capabilites to allow reusage of common snippet of code, such as `@BeforeClass` annotation that can be used over a non-static method, as if with JUnit, the same annotation must be over a static method, causing lowering of performance with the workaround `@Before`, that starts every time before `@Test` method.
#### 2.3.2 Designed to Scale
The usage of **Spring Cloud** and **Docker** allows the project to be scaled. When more than one of the same microservice must be in strategic regions, it's just to deploy the same *docker* image in another region. And those microservices will consume the *Spring Cloud Eureka* server to get address according to their strategic regions, in a more complex configuration.
#### 2.3.3 Data & IT Security
For *data security*, error handling was planned to avoid to indirectly inform the attackers what is the Service behavior, and the *Route Service* was secured, using a simple *user* and *password* just for demonstration, as we have more robust technology such as OAuth2, JWT, etc. At least, CORS was used to filter URL origin.
#### 2.3.4 Design for Failure
As above mentioned about security side, now for the error side, all relevant Exceptions is treated with specific message, and then the irrelevant for client, it's omitted with a generic message.   

 ### 2.4 Clean Code
 I used the `@Lombok` Framework to let code cleaner, avoiding boilerplate Vanilla Java Code. Projects were organized by Domain Driven Design

## 3. Click to see the [Pipeline Proposal Slide](https://github.com/deafjava/adidas_docker/blob/master/Adidas_Microservice_Slide.png)
