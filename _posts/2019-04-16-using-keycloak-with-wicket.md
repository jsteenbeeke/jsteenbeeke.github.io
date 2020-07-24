---
layout: post
title:  "Using Keycloak with Wicket"
author: "Jeroen Steenbeeke"
date:   2019-04-16 11:28:00 +0100
---
I have a number of [Wicket](http://wicket.apache.org/)-based web applications for various hobby purposes. 
These applications each serve a specific task, such as keeping track of my financial situation, or to
inform me when my favorite YouTube channels have updates I'm interested in. A common feature among these
applications is that they have some form of identity management, usually in the form of usernames and
passwords.

Having created a number of these login systems over the years, I find the process to be quite tedious. I
considered creating my own centralized identity management system to combine the login logic for all these
applications when I heard [Adam Bien](http://adambien.blog/) mention [Keycloak](https://www.keycloak.org/) 
in one of his [Airhacks.tv](http://airhacks.tv/) episodes.

Intrigued, I did some research, and decided to use Keycloak as identity management solution for my next
project, which we'll refer to as `keycloak-client` in this post.

The application is bundled as a pair of WAR files, one for the Wicket-based frontend, and one for the
Spring and Resteasy-based backend.

<!--more-->

## Configuration

Keycloak runs as a Docker container on my home server, and on this installation I created a realm named 
"Personal" with a client named `keycloak-client` which connects using [OpenID Connect](https://openid.net/connect/). Once
this client was configured, I downloaded the `keycloak.json` file from the Installation tab in the client view, which
looked something like this:

```json

{
  "realm": "Personal",
  "auth-server-url": "https://host/path/to/keycloak",
  "ssl-required": "external",
  "resource": "keycloak-client",
  "public-client": true,
  "verify-token-audience": true,
  "use-resource-role-mappings": true,
  "confidential-port": 0
}

```
This file was placed in the `WEB-INF` folder of both the backend and the frontend. I then added
the Keycloak servlet filter to the `pom.xml` of both projects:

```xml

<dependency>
    <groupId>org.keycloak</groupId>
    <artifactId>keycloak-servlet-filter-adapter</artifactId>
    <version>3.1.0.Final</version>
</dependency>

```

I then used the following configuration for the REST backend:

```xml
        <filter>
                <filter-name>keycloak</filter-name>
                <filter-class>org.keycloak.adapters.servlet.KeycloakOIDCFilter</filter-class>
                <init-param>
                        <param-name>keycloak.config.skipPattern</param-name>
                        <param-value>/health</param-value>
                </init-param>
        </filter>
        <filter-mapping>
                <filter-name>keycloak</filter-name>
                <url-pattern>/*</url-pattern>
        </filter-mapping>

```

And the following for the Wicket frontend (which has no `/health` service):

```xml
        <filter>
                <filter-name>keycloak</filter-name>
                <filter-class>org.keycloak.adapters.servlet.KeycloakOIDCFilter</filter-class>
        </filter>
        <filter-mapping>
                <filter-name>keycloak</filter-name>
                <url-pattern>/*</url-pattern>
        </filter-mapping>

```

With this configuration, all URLs (save for the `/health` endpoint on the backend) require a valid Keycloak
session, and will automatically redirect you to Keycloak if you're not logged in.

## Accessing Keycloak's session data from Wicket

Once you're logged in, you're probably going to want to know important things such as the user's ID, and
possibly what roles the user has. This information is stored as part of the servlet request, in a class
 of type `KeycloakSecurityContext`. Accessing this class from Wicket is simply a matter of getting the current
 request from the `RequestCycle`:
 
 ```java
 
ServletWebRequest request = (ServletWebRequest) RequestCycle.get().getRequest();
HttpServletRequest containerRequest = request.getContainerRequest();
KeycloakSecurityContext securityContext = (KeycloakSecurityContext) containerRequest.getAttribute(KeycloakSecurityContext.class.getName());

```

From the `KeycloakSecurityContext` you can get the `AccessToken`, which contains the User ID in the Subject property:

```java

AccessToken token = securityContext.getToken();
String subject = token.getSubject();

```

In addition to this data, the `KeycloakSecurityContext` contains a wealth of other information, which
I won't cover in detail. The most important information from a Wicket point of view is probably
the roles a user has, which can be accessed as follows:

```java

// This will probably differ for your implementation. Check field "resource" in keycloak.json
final String CLIENT_ID = "keycloak-client";

AccessToken token = securityContext.getToken();
Map<String, Access> access = token.getResourceAccess();
Set<String> roles = new HashSet<>();
if (access.containsKey(CLIENT_ID)) { 
	Set<String> userRoles = access.get(CLIENT_ID).getRoles();
	if (userRoles != null) {
		roles.addAll(userRoles);
	}
}

```

You can then use these roles to define access to various pages, perhaps using a homebrew solution
 that uses an `IComponentInstantiationListener`, or by adapting an existing solution such as 
 [wicket-auth-roles](https://ci.apache.org/projects/wicket/guide/6.x/guide/security.html) (though it might
 be easier to only partially use the auth-roles functionality since your application will not have its
 own login page). 




## Accessing Keycloak's session data from JAX-RS

The same logic can be used from a JAX-RS resource, by extracting the `KeycloakSecurityContext` from
the HttpServletRequest:

```java

@Path("/example")
public class ExampleResource { 
    
	@Context
    private HttpServletRequest servletRequest;

    @GET
    public Response whoAmI() {
        KeycloakSecurityContext context = (KeycloakSecurityContext) servletRequest.getAttribute(KeycloakSecurityContext.class);
        
        if (context == null) {
        	return Response.status(Response.Status.FORBIDDEN).build();
        }
        
        return Response.ok(context.getToken().getSubject()).build();

    }
}

```

## Accessing the backend from the frontend

My frontend communicates with my backend through REST, using `ResteasyClient` to reuse the JAX-RS interfaces
 the backend also uses. Since the backend is also secured using Keycloak, you need to pass the access token
 as header with every request you make to the backend. This can be done using a `ClientRequestFilter`:
 
```java

public class KeycloakTokenFilter implements ClientRequestFilter {

	@Override
	public void filter(ClientRequestContext requestContext) throws IOException {
	    ServletWebRequest request = (ServletWebRequest) RequestCycle.get().getRequest();
        HttpServletRequest containerRequest = request.getContainerRequest();
        KeycloakSecurityContext securityContext = (KeycloakSecurityContext) containerRequest.getAttribute(KeycloakSecurityContext.class.getName());
        
        requestContext.getHeaders().add("Authorization", "Bearer "+ securityContext.getTokenString());	
	}
}

```  

This filter is then added to the `ResteasyClient` with the register method:

```java

        ResteasyClientBuilder builder = new ResteasyClientBuilder();
		ResteasyClient client = builder.build();
		client.register(KeycloakTokenFilter.class);

		ResteasyWebTarget target = client.target(BACKEND_URL);

```

## In conclusion

The amount of configuration needed to setup Keycloak authentication is minimal, and as such is a very
useful solution for securing your Wicket applications.
