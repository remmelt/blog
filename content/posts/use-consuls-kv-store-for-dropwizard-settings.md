+++
draft = false
date = 2015-06-09T20:18:00Z
title = "Use Consul's KV store for Dropwizard settings"
description = ""
slug = ""
tags = ["dropwizard", "code", "consul", "settings", "config", "java"]
categories = []
externalLink = ""
series = []
featured_image = "images/posts/consul-logo-grad.png"
+++

![Consul logo](consul-logo-grad.png)

I wanted to see how hard it would be to get [Dropwizard](http://www.dropwizard.io/) config from [Consul’s](https://consul.io/docs/agent/http/kv.html) key value store. Turns out it is very easy to do!

Make sure you’re using Dropwizard >= 0.8.0, this has the `SubstitutingSourceProvider` that we’re going to use.

Create a default Dropwizard project with the following configuration line in its `config.yml`:

`someSetting: ${settings/someSetting}`

Don’t forget to add it to the `xxxConfiguration`:
```java
@JsonProperty
@NotEmpty
private String someSetting;
```

Now in the xxxApplication class, register a new `ConsulKVSubstitutor` in the `initialize` method:
```java
@Override
public void initialize(Bootstrap<ConsulConfigExampleConfiguration> bootstrap) {
	bootstrap.setConfigurationSourceProvider(
			new SubstitutingSourceProvider(
					bootstrap.getConfigurationSourceProvider(),
					new ConsulKVSubstitutor(false)
			)
	);
}
```

The setting with path `settings/someSetting` while be looked up in the KV store and will be replaced in the `config.yml`, after which the app will resume startup.

Lookup is done using [Ecwid’s Consul API Java bindings](https://github.com/Ecwid/consul-api).

Note that this does not register changes made to the settings in the KV store. This could be added by using a [watch](https://www.consul.io/docs/agent/watches.html) but it doesn’t look like that is currently supported in this Java lib.

[Github repo with example project](https://github.com/remmelt/dropwizard-consul-config-provider).
