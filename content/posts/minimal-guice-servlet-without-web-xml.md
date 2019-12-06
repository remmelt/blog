+++
draft = true
date = 2014-04-14T08:05:00Z
title = "Minimal Guice Servlet without web.xml"
description = ""
slug = ""
tags = []
categories = []
externalLink = ""
series = []
+++
Here are instructions and a sample project to help you set up a minimal Guice servlet project without web.xml. The sample project can be found at [github.com/remmelt/guice-web-servlet under the minimal-setup tag](https://github.com/remmelt/guice-web-servlet/tree/minimal-setup). Running it is as easy as cloning the repository, `mvn package` and deploying the `.war` file in the `target` directory to a servlet 3.0 application server.

## Dependencies
Let’s start with the `pom.xml`. We’ll need the following dependencies:
```xml
<dependencies>
    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>javax.servlet-api</artifactId>
        <version>3.0.1</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>com.google.inject</groupId>
        <artifactId>guice</artifactId>
        <version>3.0</version>
    </dependency>
    <dependency>	<groupId>com.google.inject.extensions</groupId>
        <artifactId>guice-servlet</artifactId>
        <version>3.0</version>
    </dependency>
</dependencies>
```
## WebListener
Next is the web listener. This is where the Guice injector and the ServletModule are created. According to the [Guice documentation](https://code.google.com/p/google-guice/wiki/ServletModule), the listener is the logical place to configure binding. I have done so on line 10, where I’m binding the MessageSenderImpl implementation to the MessageSender interface.
```java
@WebListener
public class GuiceServletConfig extends GuiceServletContextListener {
	@Override
	protected Injector getInjector() {
		return Guice.createInjector(new ServletModule() {
			@Override
			protected void configureServlets() {
				super.configureServlets();
				serve("/*").with(WiredServlet.class);
				bind(MessageSender.class).to(MessageSenderImpl.class);
			}
		});
	}
}
```
## WebFilter
Note that you can also add filters in the configureServlets method like this: `filter("/*").through(MyFilter.class);` Somehow these filters were not picked up by my webapp, so I created a separate WebFilter-annotated class. See below.
```java
@WebFilter("/*")
public class GuiceWebFilter extends GuiceFilter{
	@Override
	public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
		super.doFilter(servletRequest, servletResponse, filterChain);
	}
}
```
## The Servlet
I have configured the ServletModule to serve anything `"/*"` with the `WiredServlet` class. This class needs to be a singleton. Of note in this class is the injected `MessageSender` service interface. The messageSender object will get bound by the `MessageSenderImpl` class due to the `bind(MessageSender.class).to(MessageSenderImpl.class);` line in the `WebListener`.
```java
@Singleton
public class WiredServlet extends HttpServlet {
	@Inject
	private MessageSender messageSender;

	@Override
	protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
		resp.getOutputStream().print(messageSender.sendMessage("Hello world!"));
	}
}
```
## Service and implementation
Finally we’ll need a service interface and its implementation.
```java
public interface MessageSender {
	String sendMessage(String msg);
}
```
```java
public class MessageSenderImpl implements MessageSender {
	@Override
	public String sendMessage(String msg) {
		return "Message sent: " + msg;
	}
}
```
## Result
Package the project using `mvn clean package` from the command line. This will result in a `.war` file in the `target` directory. Deploy this to any `servlet-api 3.0` application server, I’m using Tomcat 7.0.x. Visit the resulting site, any URL will be picked up by `"/*"`. For Tomcat, try visiting localhost:8080. Any page should display `Message sent: Hello world!` The `Hello world!` message was “sent” by the bound `MessageSenderImpl` implementation.

Now start binding parameters!
