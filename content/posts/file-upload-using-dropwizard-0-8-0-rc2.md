+++
draft = false
date = 2015-02-15T21:27:00Z
title = "File upload using Dropwizard 0.8.0-rc2"
description = ""
slug = ""
tags = ["dropwizard", "jersey", "code", "multipart"]
categories = []
externalLink = ""
series = []
+++

Dropwizard 0.7.1 used `com.sun.jersey:jersey-server:jar:1.18.1`, where version 0.8.0 comes with `org.glassfish.jersey.core:jersey-server:jar:2.15`.

Before, when uploading files, it was enough to add the `com.sun.jersey.contribs:jersey-multipart` dependency like so:

```xml
<dependency>
	<groupId>com.sun.jersey.contribs</groupId>
	<artifactId>jersey-multipart</artifactId>
	<version>1.18.3</version>
</dependency>
```

This will not work anymore and throws the somewhat cryptic exception: `java.lang.IllegalArgumentException: The MultiPartConfig instance we expected is not present. Have you registered the MultiPartConfigProvider class?`

The correct dependency should be

```xml
<dependency>
	<groupId>org.glassfish.jersey.media</groupId>
	<artifactId>jersey-media-multipart</artifactId>
	<version>2.15</version>
</dependency>
```
Update your imports and you’re good to go. The same resource will suffice.
```java
@Timed
@POST
@Consumes(MediaType.MULTIPART_FORM_DATA)
public void fileUploaded(@FormDataParam("file") final InputStream inputStream,
    @FormDataParam("file") final FormDataContentDisposition contentDispositionHeader) {
	storeImage(inputStream);
}
```
Don’t forget to register the MultiPartFeature in your Application’s `run` method:

```java
environment.jersey().register(MultiPartFeature.class);
```
