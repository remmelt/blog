+++
draft = false
date = 2014-09-14T21:36:00Z
title = "Basic Dropwizard example"
description = ""
slug = ""
tags = []
categories = ["dropwizard", "microservice"]
externalLink = ""
series = []
+++
I’ve been trying out [Dropwizard](https://dropwizard.github.io/dropwizard/) the last couple of months and it’s been very nice. There are many, many frameworks to choose from in Java country, so it comes as a relief to have a bunch of them chosen for me. Dropwizard glues Jersey, Jetty, Jackson and more together. The end result is a reduction in boiler plate code and complexity. Great for micro services.

I am going to write a couple of posts about Dropwizard. The next one will add a secured resource using OAuth2, and there will also be an example using JWT.

## Goal
This is an example of a very basic Dropwizard resource. The goal is to have a simple, tested API endpoint on `/ping` that returns `pong`.

##
First steps
Clone the example repository by running
```bash
git clone https://github.com/remmelt/dropwizard-ping.git
```
and checking out tag “ping-resource” or you can browse the code at [GitHub](https://github.com/remmelt/dropwizard-ping).

First things first. Create a banner.txt in `src/main/resources`. This banner will be displayed when you start up your Dropwizard application, so make it look nice. Now we have that important step out of the way, let’s move on.

The `pom.xml` is straightforward, following suggestions in the docs about the shade plugin.

## The Application
The `PingApplication` is quite empty, it just registers the Ping resource and a single health check so Dropwizard doesn’t complain on start up.

Next is the configuration class `PingConfiguration`. Also empty for now.

Note that the `config.yml` in `src/main/resources` contains a redundant logging configuration. This is so Dropwizard picks up the file as YAML instead of JSON. If you don’t do this, the application will throw a `NullPointerException` on startup without any clue as to what might be the problem.

## Ping resource
The interesting bits are in the `PingResource` class. This doesn’t do much currently, it just returns a small JSON string.
```java
package com.remmelt.examples.resources;

import com.codahale.metrics.annotation.Timed;
import lombok.AllArgsConstructor;

import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

@Path("/ping")
@Produces(MediaType.APPLICATION_JSON)
public class PingResource {
	@GET
	@Timed
	public String pong() {
		return "{\"answer\": \"pong\"}";
	}
}
```
Package the app with `mvn clean package` and run by issueing `java -jar target/dropwizard-ping-1.0-SNAPSHOT.jar server src/main/resources/config.yml`.

You can now browse to localhost:8080/ping or run curl [localhost:8080/ping](http://localhost:8080/ping). I would also suggest installing [HTTPie](https://github.com/jakubroztocil/httpie), makes life a lot easier (and prettier!)

```
% http localhost:8080/ping
HTTP/1.1 200 OK
Content-Encoding: gzip
Content-Type: application/json
Date: Fri, 04 Jul 2014 22:59:43 GMT
Transfer-Encoding: chunked
Vary: Accept-Encoding

{
    "answer": "pong"
}
```
Note that you can interact with the Dropwizard application on port [8081](http://localhost:8081/) as well. This is the admin interface that returns json-encoded information about the state of the app. Earlier, I’ve annotated the `pong` resource method with [`@Timed`](https://dropwizard.github.io/dropwizard/manual/core.html#metrics). We can now track interesting statistics on the number of calls and timing of the method invocation under the metrics header.

## Conclusion
As promised, it’s very easy to get going with Dropwizard. Most boiler plate code has been taken care of by the framework and there are already some interesting things we can do.
