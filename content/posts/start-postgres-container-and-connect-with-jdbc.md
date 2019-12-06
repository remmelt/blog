+++
draft = false
date = 2015-01-14T22:57:00Z
title = "Start Postgres container and connect with JDBC"
tags = ["postgrtes", "docker", "spotify", "docker"]
+++
Quick example of how to set up a Docker container with Postgresql, using the [Spotify Docker Java client](https://github.com/spotify/docker-client/).

This supposes a running Docker installation on your local computer, for example using boot2docker.

```java

import com.google.common.base.Strings;
import com.google.common.net.HostAndPort;
import com.spotify.docker.client.DefaultDockerClient;
import com.spotify.docker.client.DockerCertificateException;
import com.spotify.docker.client.DockerClient;
import com.spotify.docker.client.DockerException;
import com.spotify.docker.client.messages.ContainerConfig;
import com.spotify.docker.client.messages.ContainerCreation;
import com.spotify.docker.client.messages.ContainerInfo;
import com.spotify.docker.client.messages.HostConfig;
import lombok.extern.slf4j.Slf4j;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

@Slf4j
public abstract class PostgresContainerExample {

	public static Connection setUpDbContainer() throws SQLException {
		try {
			// This will only work with the DOCKER_HOST environment variable set
			final DockerClient docker = DefaultDockerClient.fromEnv().build();
			final ContainerConfig config = ContainerConfig.builder()
					.image("postgres:9.3")
					.build();
			final ContainerCreation creation = docker.createContainer(config);
			final String id = creation.id();

			final HostConfig hostConfig = HostConfig.builder()
					.publishAllPorts(true)
					.build();

			// Container is now created, let's start it up
			docker.startContainer(id, hostConfig);

			// startContainer swallows errors, so check if the container is in the running state
			final ContainerInfo info = docker.inspectContainer(id);
			if (!info.state().running()) {
				throw new IllegalStateException("Could not start Postgres container");
			}

			// We need to build a connection string to connect to Postgres
			// Fiddly way to find the host, relies on the environment variable.
			final String endpoint = System.getenv("DOCKER_HOST");
			final HostAndPort hostAndPort = HostAndPort.fromString(endpoint.replaceAll(".*://", ""));
			final String hostText = hostAndPort.getHostText();
			final String address = Strings.isNullOrEmpty(hostText) ? "127.0.0.1" : hostText;

			// Find the random port in the network settings
			final int port = Integer.valueOf(info.networkSettings().ports().get("5432/tcp").get(0).hostPort());

			final String connectionString = String.format("jdbc:postgresql://%s:%d/postgres?user=postgres", address, port);

			// It takes a while for the Postgres application to start up inside the container. We're going to give it 5 seconds.
			Connection conn = null;
			int tries = 1;
			while (conn == null && tries <= 10) {
				try {
					conn = DriverManager.getConnection(connectionString);
				} catch (SQLException ignored) {
					tries++;
					log.debug("Retrying ({}/10)...", tries);

					Thread.sleep(500);
				}
			}
			return conn;
		} catch (DockerCertificateException | InterruptedException | DockerException e) {
			e.printStackTrace();
		}
		throw new SQLException("Could not connect to Postgres container");
	}
}
```