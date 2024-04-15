---
layout: post
title: "Spring Boot and OAuth2"
author: "Tejaswi Goudru"
meta: "SpringBoot"
---
Reference: [https://www.journaldev.com/1330/java-collections-interview-questions-and-answers](https://spring.io/guides/tutorials/spring-boot-oauth2)

Spring Boot and OAuth2
======================

This guide shows you how to build a sample app doing various things with "social login" using [OAuth 2.0](https://tools.ietf.org/html/rfc6749) and [Spring Boot](https://projects.spring.io/spring-boot/).

It starts with a simple, single-provider single-sign on, and works up to a client with a choice of authentication providers: [GitHub](https://github.com/settings/developers) or [Google](https://developers.google.com/identity/protocols/OpenIDConnect).

The samples are all single-page apps using Spring Boot and Spring Security on the back end. They also all use plain [jQuery](https://jquery.org/) on the front end. But, the changes needed to convert to a different JavaScript framework or to use server-side rendering would be minimal.

All samples are implemented using the native OAuth 2.0 support in [Spring Boot](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-security-oauth2).

There are several samples building on each other, adding new features at each step:

*   [**simple**](#_social_login_simple): a very basic static app with just a home page and unconditional login via Spring Boot’s OAuth 2.0 configuration properties (if you visit the home page, you will be automatically redirected to GitHub).
    
*   [**click**](#_social_login_click): adds an explicit link that the user has to click to login.
    
*   [**logout**](#_social_login_logout): adds a logout link as well for authenticated users.
    
*   [**two-providers**](#_social_login_two_providers): adds a second login provider so the user can choose on the home page which one to use.
    
*   [**custom-error**](#_social_login_custom_error): adds an error message for unauthenticated users, and a custom authentication based on GitHub’s API.
    

The changes needed to migrate from one app to the next one in the feature ladder can be tracked in [the source code](https://github.com/spring-guides/tut-spring-boot-oauth2). Each version of the app is its own directory so that you can compare their differences.

Each app can be imported into an IDE. You can run the `main` method in `SocialApplication` to start an app. They all come up with a home page on [http://localhost:8080](http://localhost:8080) (and all require that you have at least a GitHub and Google account if you want to log in and see the content).

You can also run all the apps on the command line using `mvn spring-boot:run` or by building the jar file and running it with `mvn package` and `java -jar target/*.jar` (per the [Spring Boot docs](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#getting-started-first-application-run) and other [available documentation](https://spring.io/guides/gs/spring-boot/)). There is no need to install Maven if you use the [wrapper](https://github.com/takari/maven-wrapper) at the top level, e.g.

    CopyCopy$ cd simple
    $ ../mvnw package
    $ java -jar target/*.jar

The apps all work on `localhost:8080` because they’ll use OAuth 2.0 clients registered with GitHub and Google for that address. To run them on a different host or port, you need to register your apps that way. There is no danger of leaking your credentials beyond localhost if you use the default values. But, be careful what you expose on the Internet, and don’t put your own app registrations in public source control.

Single Sign On With GitHub
--------------------------

In this section, you’ll create a minimal application that uses GitHub for authentication. This will be quite easy by taking advantage of the autoconfiguration features in Spring Boot.

### Creating a New Project

First, you need to create a Spring Boot application, which can be done in a number of ways. The easiest is to go to [https://start.spring.io](https://start.spring.io) and generate an empty project (choosing the "Web" dependency as a starting point). Equivalently, do this on the command line:

    CopyCopy$ mkdir ui && cd ui
    $ curl https://start.spring.io/starter.tgz -d style=web -d name=simple | tar -xzvf -

You can then import that project into your favorite IDE (it’s a normal Maven Java project by default), or just work with the files and `mvn` on the command line.

### Add a Home Page

In your new project, create `index.html` in the `src/main/resources/static` folder. You should add some stylesheets and JavaScript links so the result looks like this:

index.html

    CopyCopy<!doctype html>
    <html lang="en">
    <head>
        <meta charset="utf-8"/>
        <meta http-equiv="X-UA-Compatible" content="IE=edge"/>
        <title>Demo</title>
        <meta name="description" content=""/>
        <meta name="viewport" content="width=device-width"/>
        <base href="/"/>
        <link rel="stylesheet" type="text/css" href="/webjars/bootstrap/css/bootstrap.min.css"/>
        <script type="text/javascript" src="/webjars/jquery/jquery.min.js"></script>
        <script type="text/javascript" src="/webjars/bootstrap/js/bootstrap.min.js"></script>
    </head>
    <body>
    	<h1>Demo</h1>
    	<div class="container"></div>
    </body>
    </html>

None of this is necessary to demonstrate the OAuth 2.0 login features, but it’ll be nice to have a pleasant UI in the end, so you might as well start with some basic stuff in the home page.

If you start the app and load the home page, you’ll notice that the stylesheets have not been loaded. So, you need to add those as well by adding jQuery and Twitter Bootstrap:

pom.xml

    CopyCopy<dependency>
    	<groupId>org.webjars</groupId>
    	<artifactId>jquery</artifactId>
    	<version>3.4.1</version>
    </dependency>
    <dependency>
    	<groupId>org.webjars</groupId>
    	<artifactId>bootstrap</artifactId>
    	<version>4.3.1</version>
    </dependency>
    <dependency>
    	<groupId>org.webjars</groupId>
    	<artifactId>webjars-locator-core</artifactId>
    </dependency>

The final dependency is the webjars "locator" which is provided as a library by the webjars site. Spring can use the locator to locate static assets in webjars without needing to know the exact versions (hence the versionless `/webjars/**` links in the `index.html`). The webjar locator is activated by default in a Spring Boot app, as long as you don’t switch off the MVC autoconfiguration.

With those changes in place, you should have a nice looking home page for your app.

### Securing the Application with GitHub and Spring Security

To make the application secure, you can simply add Spring Security as a dependency. Since you’re wanting to do a "social" login (delegate to GitHub), you should include the Spring Security OAuth 2.0 Client starter:

pom.xml

    CopyCopy<dependency>
    	<groupId>org.springframework.boot</groupId>
    	<artifactId>spring-boot-starter-oauth2-client</artifactId>
    </dependency>

By adding that, it will secure your app with OAuth 2.0 by default.

Next, you need to configure your app to use GitHub as the authentication provider. To achieve this, do the following:

*   [Add a New GitHub app](#github-register-application)
    
*   [Configure application.yml](#github-application-config)
    
*   [Boot up the application](#github-boot-application)
    

#### Add a New GitHub App

To use GitHub’s OAuth 2.0 authentication system for login, you must first [Add a new GitHub app](https://github.com/settings/developers).

Select "New OAuth App" and then the "Register a new OAuth application" page is presented. Enter an app name and description. Then, enter your app’s home page, which should be [http://localhost:8080](http://localhost:8080), in this case. Finally, indicate the Authorization callback URL as `[http://localhost:8080/login/oauth2/code/github](http://localhost:8080/login/oauth2/code/github)` and click _Register Application_.

The OAuth redirect URI is the path in the application that the end-user’s user-agent is redirected back to after they have authenticated with GitHub and have granted access to the application on the _Authorize application_ page.

The default redirect URI template is `{baseUrl}/login/oauth2/code/{registrationId}`. The **_registrationId_** is a unique identifier for the `ClientRegistration`.

#### Configure `application.yml`

Then, to make the link to GitHub, add the following to your `application.yml`:

application.yml

    CopyCopyspring:
      security:
        oauth2:
          client:
            registration:
              github:
                clientId: github-client-id
                clientSecret: github-client-secret
    # ...

Simply use the OAuth 2.0 credentials you just created with GitHub, replacing `github-client-id` with the client id and `github-client-secret` with the client secret.

##### Boot Up the Application

With that change, you can run your app again and visit the home page at [http://localhost:8080](http://localhost:8080). Now, instead of the home page, you should be redirected to login with GitHub. If you do that, and accept any authorizations you are asked to make, you will be redirected back to the local app, and the home page will be visible.

If you stay logged in to GitHub, you won’t have to re-authenticate with this local app, even if you open it in a fresh browser with no cookies and no cached data. (That’s what Single Sign-On means.)

If you are working through this section with the sample application, be sure to clear your browser cache of cookies and HTTP Basic credentials. The best way to do that for a single server is to open a new private window.

It’s safe to grant access to this sample since only the app running locally can use the tokens and the scope it asks for is limited. Be aware of what you are approving when you log into apps like this though: They might ask for permission to do more than you are comfortable with (e.g. they might ask for permission to change your personal data, which is unlikely to be in your interest).

### What Just Happened?

The app you just wrote, in OAuth 2.0 terms, is a _Client Application_, and it uses the [authorization code grant](https://tools.ietf.org/html/rfc6749#section-4) to obtain an access token from GitHub (the Authorization Server).

It then uses the access token to ask GitHub for some personal details (only what you permitted it to do), including your login ID and your name. In this phase, GitHub is acting as a Resource Server, decoding the token that you send and checking if it gives the app permission to access the user’s details. If that process is successful, the app inserts the user details into the Spring Security context so that you are authenticated.

If you look in the browser tools (F12 on Chrome or Firefox) and follow the network traffic for all the hops, you will see the redirects back and forth with GitHub, and finally you’ll land back on the home page with a new `Set-Cookie` header. This cookie (`JSESSIONID` by default) is a token for your authentication details for Spring (or any servlet-based) applications.

So we have a secure application, in the sense that to see any content a user has to authenticate with an external provider (GitHub).

We wouldn’t want to use that for an internet banking website. But for basic identification purposes, and to segregate content between different users of your site, it’s an excellent starting point. That’s why this kind of authentication is very popular these days.

In the next section, we are going to add some basic features to the application. We’ll also make it a bit more obvious to users what is going on when they get that initial redirect to GitHub.

Add a Welcome Page
------------------

In this section, you’ll modify the [simple](#_social_login_simple) app you just built by adding an explicit link to login with GitHub. Instead of being redirected immediately, the new link will be visible on the home page, and the user can choose to login or to stay unauthenticated. Only when the user has clicked on the link will the secure content be rendered.

### Conditional Content on the Home Page

To render content on the condition that the user is authenticated, you have the option of either server-side or client-side rendering.

Here, you’ll change the client side with [JQuery](https://jquery.org/), though if you prefer to use something else, it shouldn’t be very hard to translate the client code.

To get started with the dynamic content, you need to mark a couple of HTML elements like so:

index.html

    CopyCopy<div class="container unauthenticated">
        With GitHub: <a href="/oauth2/authorization/github">click here</a>
    </div>
    <div class="container authenticated" style="display:none">
        Logged in as: <span id="user"></span>
    </div>

By default, the first `<div>` will show, and the second one won’t. Note also the empty `<span>` with an `id` attribute.

In a moment, you’ll add a server-side endpoint that will return the logged in user details as JSON.

But, first, add the following JavaScript, which will hit that endpoint. Based on the endpoint’s response, this JavaScript will populate the `<span>` tag with the user’s name and toggle the `<div>` appropriately:

index.html

    CopyCopy<script type="text/javascript">
        $.get("/user", function(data) {
            $("#user").html(data.name);
            $(".unauthenticated").hide()
            $(".authenticated").show()
        });
    </script>

Note that this JavaScript expects the server-side endpoint to be called `/user`.

### The `/user` Endpoint

Now, you’ll add the server-side endpoint just mentioned, calling it `/user`. It will send back the currently logged-in user, which we can do quite easily in our main class:

SocialApplication.java

    CopyCopy@SpringBootApplication
    @RestController
    public class SocialApplication {
    
        @GetMapping("/user")
        public Map<String, Object> user(@AuthenticationPrincipal OAuth2User principal) {
            return Collections.singletonMap("name", principal.getAttribute("name"));
        }
    
        public static void main(String[] args) {
            SpringApplication.run(SocialApplication.class, args);
        }
    
    }

Note the use of `@RestController`, `@GetMapping`, and the `OAuth2User` injected into the handler method.

It’s not a great idea to return a whole `OAuth2User` in an endpoint since it might contain information you would rather not reveal to a browser client.

### Making the Home Page Public

There’s one final change you’ll need to make.

This app will now work fine and authenticate as before, but it’s still going to redirect before showing the page. To make the link visible, we also need to switch off the security on the home page by extending `WebSecurityConfigurerAdapter`:

SocialApplication

    CopyCopy@SpringBootApplication
    @RestController
    public class SocialApplication extends WebSecurityConfigurerAdapter {
    
        // ...
    
        @Override
        protected void configure(HttpSecurity http) throws Exception {
        	// @formatter:off
            http
                .authorizeRequests(a -> a
                    .antMatchers("/", "/error", "/webjars/**").permitAll()
                    .anyRequest().authenticated()
                )
                .exceptionHandling(e -> e
                    .authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED))
                )
                .oauth2Login();
            // @formatter:on
        }
    
    }

Spring Boot attaches special meaning to a `WebSecurityConfigurerAdapter` on the class annotated with `@SpringBootApplication`: It uses it to configure the security filter chain that carries the OAuth 2.0 authentication processor.

The above configuration indicates a whitelist of permitted endpoints, with every other endpoint requiring authentication.

You want to allow:

*   `/` since that’s the page you just made dynamic, with some of its content visible to unauthenticated users
    
*   `/error` since that’s a Spring Boot endpoint for displaying errors, and
    
*   `/webjars/**` since you’ll want your JavaScript to run for all visitors, authenticated or not
    

You won’t see anything about `/user` in this configuration, though. Everything, including `/user` remains secure unless indicated because of the `.anyRequest().authenticated()` configuration at the end.

Finally, since we are interfacing with the backend over Ajax, we’ll want to configure endpoints to respond with a 401 instead of the default behavior of redirecting to a login page. Configuring the `authenticationEntryPoint` achieves this for us.

With those changes in place, the application is complete, and if you run it and visit the home page you should see a nicely styled HTML link to "login with GitHub". The link takes you not directly to GitHub, but to the local path that processes the authentication (and sends a redirect to GitHub). Once you have authenticated, you get redirected back to the local app, where it now displays your name (assuming you have set up your permissions in GitHub to allow access to that data).

Add a Logout Button
-------------------

In this section, we modify the [click](#_social_login_click) app we built by adding a button that allows the user to log out of the app. This seems like a simple feature, but it requires a bit of care to implement, so it’s worth spending some time discussing exactly how to do it. Most of the changes are to do with the fact that we are transforming the app from a read-only resource to a read-write one (logging out requires a state change), so the same changes would be needed in any realistic application that wasn’t just static content.

### Client Side Changes

On the client, we just need to provide a logout button and some JavaScript to call back to the server to ask for the authentication to be cancelled. First, in the "authenticated" section of the UI, we add the button:

index.html

<div class="container authenticated">
  Logged in as: <span id="user"></span>
  <div>
    <button onClick="logout()" class="btn btn-primary">Logout</button>
  </div>
</div>

and then we provide the `logout()` function that it refers to in the JavaScript:

index.html

var logout = function() {
    $.post("/logout", function() {
        $("#user").html('');
        $(".unauthenticated").show();
        $(".authenticated").hide();
    })
    return true;
}

The `logout()` function does a POST to `/logout` and then clears the dynamic content. Now we can switch over to the server side to implement that endpoint.

### Adding a Logout Endpoint

Spring Security has built in support for a `/logout` endpoint which will do the right thing for us (clear the session and invalidate the cookie). To configure the endpoint we simply extend the existing `configure()` method in our `WebSecurityConfigurerAdapter`:

SocialApplication.java

    CopyCopy@Override
    protected void configure(HttpSecurity http) throws Exception {
    	// @formatter:off
        http
            // ... existing code here
            .logout(l -> l
                .logoutSuccessUrl("/").permitAll()
            )
            // ... existing code here
        // @formatter:on
    }

The `/logout` endpoint requires us to POST to it, and to protect the user from Cross Site Request Forgery (CSRF, pronounced "sea surf"), it requires a token to be included in the request. The value of the token is linked to the current session, which is what provides the protection, so we need a way to get that data into our JavaScript app.

Many JavaScript frameworks have built in support for CSRF (e.g. in Angular they call it XSRF), but it is often implemented in a slightly different way than the out-of-the box behaviour of Spring Security. For instance, in Angular, the front end would like the server to send it a cookie called "XSRF-TOKEN" and if it sees that, it will send the value back as a header named "X-XSRF-TOKEN". We can implement the same behaviour with our simple jQuery client, and then the server-side changes will work with other front end implementations with no or very few changes. To teach Spring Security about this we need to add a filter that creates the cookie.

In the `WebSecurityConfigurerAdapter` we do the following:

SocialApplication.java

    CopyCopy@Override
    protected void configure(HttpSecurity http) throws Exception {
    	// @formatter:off
        http
            // ... existing code here
            .csrf(c -> c
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            )
            // ... existing code here
        // @formatter:on
    }

### Adding the CSRF Token in the Client

Since we are not using a higher level framework in this sample, you’ll need to explicitly add the CSRF token, which you just made available as a cookie from the backend. To make the code a bit simpler, include the `js-cookie` library:

pom.xml

    CopyCopy<dependency>
        <groupId>org.webjars</groupId>
        <artifactId>js-cookie</artifactId>
        <version>2.1.0</version>
    </dependency>

And then, you can reference it in your HTML:

index.html

    CopyCopy<script type="text/javascript" src="/webjars/js-cookie/js.cookie.js"></script>

Finally, you can use `Cookies` convenience methods in XHR:

index.html

    CopyCopy$.ajaxSetup({
      beforeSend : function(xhr, settings) {
        if (settings.type == 'POST' || settings.type == 'PUT'
            || settings.type == 'DELETE') {
          if (!(/^http:.*/.test(settings.url) || /^https:.*/
            .test(settings.url))) {
            // Only send the token to relative URLs i.e. locally.
            xhr.setRequestHeader("X-XSRF-TOKEN",
              Cookies.get('XSRF-TOKEN'));
          }
        }
      }
    });

### Ready To Roll!

With those changes in place, we are ready to run the app and try out the new logout button. Start the app and load the home page in a new browser window. Click on the "Login" link to take you to GitHub (if you are already logged in there you might not notice the redirect). Click on the "Logout" button to cancel the current session and return the app to the unauthenticated state. If you are curious, you should be able to see the new cookies and headers in the requests that the browser exchanges with the local server.

Remember that now the logout endpoint is working with the browser client, then all other HTTP requests (POST, PUT, DELETE, etc.) will also work just as well. So this should be a good platform for an application with some more realistic features.

Login with GitHub
-----------------

In this section, you’ll modify the [logout](#_social_login_logout) app you built already, adding a sticker page so that the end-user can choose between multiple sets of credentials.

Let’s add Google as a second option for the end user.

### Initial setup

To use Google’s OAuth 2.0 authentication system for login, you must set up a project in the Google API Console to obtain OAuth 2.0 credentials.

[Google’s OAuth 2.0 implementation](https://developers.google.com/identity/protocols/OpenIDConnect) for authentication conforms to the [OpenID Connect 1.0](https://openid.net/connect/) specification and is [OpenID Certified](https://openid.net/certification/).

Follow the instructions on the [OpenID Connect](https://developers.google.com/identity/protocols/OpenIDConnect) page, starting in the section, "Setting up OAuth 2.0".

After completing the "Obtain OAuth 2.0 credentials" instructions, you should have a new OAuth Client with credentials consisting of a Client ID and a Client Secret.

### Setting the redirect URI

Also, you’ll need to supply a redirect URI, as you did for GitHub earlier.

In the "Set a redirect URI" sub-section, ensure that the **Authorized redirect URIs** field is set to `[http://localhost:8080/login/oauth2/code/google](http://localhost:8080/login/oauth2/code/google)`.

### Adding the Client Registration

Then, you need to configure the client to point Google. Because Spring Security is built with multiple clients in mind, you can add our Google credentials alongside the ones you created for GitHub:

application.yml

    CopyCopyspring:
      security:
        oauth2:
          client:
            registration:
              github:
                clientId: github-client-id
                clientSecret: github-client-secret
              google:
                client-id: google-client-id
                client-secret: google-client-secret

As you can see, Google is another provider that Spring Security ships out-of-the-box support for.

### Adding the Login Link

In the client, the change is trivial - you can just add another link:

index.html

    CopyCopy<div class="container unauthenticated">
      <div>
        With GitHub: <a href="/oauth2/authorization/github">click here</a>
      </div>
      <div>
        With Google: <a href="/oauth2/authorization/google">click here</a>
      </div>
    </div>

The final path in the URL should match the client registration id in `application.yml`.

Spring Security ships with a default provider selection page that can be reached by pointing to `/login` instead of `/oauth2/authorization/{registrationId}`.

### How to Add a Local User Database

Many applications need to hold data about their users locally, even if authentication is delegated to an external provider. We don’t show the code here, but it is easy to do in two steps.

1.  Choose a backend for your database, and set up some repositories (using Spring Data, say) for a custom `User` object that suits your needs and can be populated, fully or partially, from external authentication.
    
2.  Implement and expose `OAuth2UserService` to call the Authorization Server as well as your database. Your implementation can delegate to the default implementation, which will do the heavy lifting of calling the Authorization Server. Your implementation should return something that extends your custom `User` object and implements `OAuth2User`.
    

Hint: add a field in the `User` object to link to a unique identifier in the external provider (not the user’s name, but something that’s unique to the account in the external provider).

Adding an Error Page for Unauthenticated Users
----------------------------------------------

In this section, you’ll modify the [two-providers](#_social_login_two_providers) app you built earlier to give some feedback to users that cannot authenticate. At the same time you’ll extend the authentication logic to include a rule that only allows users if they belong to a specific GitHub organization. The "organization" is a GitHub domain-specific concept, but similar rules could be devised for other providers. For example, with Google you might want to only authenticate users from a specific domain.

### Switching to GitHub

The [two-providers](#_social_login_two_providers) sample uses GitHub as an OAuth 2.0 provider:

application.yml

    CopyCopyspring:
      security:
        oauth2:
          client:
            registration:
              github:
                client-id: bd1c0a783ccdd1c9b9e4
                client-secret: 1a9030fbca47a5b2c28e92f19050bb77824b5ad1
              # ...

### Detecting an Authentication Failure in the Client

On the client, you might like to provide some feedback for a user that could not authenticate. To facilitate this, you can add a div to which you’ll eventually add an informative message.

index.html

<div class="container text-danger error"></div>

Then, add a call to the `/error` endpoint, populating the `<div>` with the result:

index.html

$.get("/error", function(data) {
    if (data) {
        $(".error").html(data);
    } else {
        $(".error").html('');
    }
});

The error function checks with the backend if there is any error to display

### Adding an Error Message

To support the retrieval of an error message, you’ll need to capture it when authentication fails. To achieve this, you can configure an `AuthenticationFailureHandler`, like so:

    CopyCopyprotected void configure(HttpSecurity http) throws Exception {
    	// @formatter:off
    	http
    	    // ... existing configuration
    	    .oauth2Login(o -> o
                .failureHandler((request, response, exception) -> {
    			    request.getSession().setAttribute("error.message", exception.getMessage());
    			    handler.onAuthenticationFailure(request, response, exception);
                })
            );
    }

The above will save an error message to the session whenever authentication fails.

Then, you can add a simple `/error` controller, like this one:

SocialApplication.java

    CopyCopy@GetMapping("/error")
    public String error(HttpServletRequest request) {
    	String message = (String) request.getSession().getAttribute("error.message");
    	request.getSession().removeAttribute("error.message");
    	return message;
    }

This will replace the default `/error` page in the app, which is fine for our case, but may not be sophisticated enough for your needs.

### Generating a 401 in the Server

A 401 response will already be coming from Spring Security if the user cannot or does not want to login with GitHub, so the app is already working if you fail to authenticate (e.g. by rejecting the token grant).

To spice things up a bit, you can extend the authentication rule to reject users that are not in the right organization.

You can use the GitHub API to find out more about the user, so you’ll just need to plug that into the right part of the authentication process.

Fortunately, for such a simple use case, Spring Boot has provided an easy extension point: If you declare a `@Bean` of type `OAuth2UserService`, it will be used to identify the user principal. You can use that hook to assert the the user is in the correct organization, and throw an exception if not:

SocialApplication.java

    CopyCopy@Bean
    public OAuth2UserService<OAuth2UserRequest, OAuth2User> oauth2UserService(WebClient rest) {
        DefaultOAuth2UserService delegate = new DefaultOAuth2UserService();
        return request -> {
            OAuth2User user = delegate.loadUser(request);
            if (!"github".equals(request.getClientRegistration().getRegistrationId())) {
            	return user;
            }
    
            OAuth2AuthorizedClient client = new OAuth2AuthorizedClient
                    (request.getClientRegistration(), user.getName(), request.getAccessToken());
            String url = user.getAttribute("organizations_url");
            List<Map<String, Object>> orgs = rest
                    .get().uri(url)
                    .attributes(oauth2AuthorizedClient(client))
                    .retrieve()
                    .bodyToMono(List.class)
                    .block();
    
            if (orgs.stream().anyMatch(org -> "spring-projects".equals(org.get("login")))) {
                return user;
            }
    
            throw new OAuth2AuthenticationException(new OAuth2Error("invalid_token", "Not in Spring Team", ""));
        };
    }

Note that this code is dependent on a `WebClient` instance for accessing the GitHub API on behalf of the authenticated user. Having done that, it loops over the organizations, looking for one that matches "spring-projects" (this is the organization that is used to store Spring open source projects). You can substitute your own value there if you want to be able to authenticate successfully and you are not in the Spring Engineering team. If there is no match, it throws an `OAuth2AuthenticationException`, and this is picked up by Spring Security and turned in to a 401 response.

The `WebClient` has to be created as a bean as well, but that’s trivial because its ingredients are all autowirable by virtue of having used `spring-boot-starter-oauth2-client`:

    CopyCopy@Bean
    public WebClient rest(ClientRegistrationRepository clients, OAuth2AuthorizedClientRepository authz) {
        ServletOAuth2AuthorizedClientExchangeFilterFunction oauth2 =
                new ServletOAuth2AuthorizedClientExchangeFilterFunction(clients, authz);
        return WebClient.builder()
                .filter(oauth2).build();
    }

Obviously the code above can be generalized to other authentication rules, some applicable to GitHub and some to other OAuth 2.0 providers. All you need is the `WebClient` and some knowledge of the provider’s API.
