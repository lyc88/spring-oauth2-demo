Adding an Error Page for Unauthenticated Users

In this section we modify the logout app we built earlier, switching to Github authentication, and also giving some feedback to users that cannot authenticate. At the same time we take the opportunity to extend the authentication logic to include a rule that only allows users if they belong to a specific Github organization. The "organization" is a Github domain-specific concept, but similar rules could be devised for other providers, e.g. with Google you might want to only authenticate users from a specific domain.

1、Switching to Github
  
2、Detecting an Authentication Failure in the Client
On the client we need to be able to provide some feedback for a user that could not authenticate. To facilitate this we add a div with an informative message:
``` html
<div class="container text-danger" ng-show="home.error">
There was an error (bad credentials).
</div>
```

This text will only be shown when the "error" flag is set in the controller, so we need some code to do that:

index.html
``` javascript
angular
  .module("app", [])
  .controller("home", function($http, $location) {
    var self = this;
    self.logout = function() {
      if ($location.absUrl().indexOf("error=true") >= 0) {
        self.authenticated = false;
        self.error = true;
      }
      ...
    };
  });
```

3、Adding an Error Page
To support the flag setting in the client we need to be able to capture an authentication error and redirect to the home page with that flag set in query parameters. Hence we need an endpoint, in a regular @Controller like this:
SocialApplication.java
@RequestMapping("/unauthenticated")
``` java
public String unauthenticated() {
  return "redirect:/?error=true";
}
```

In the sample app we put this in the main application class, which is now a @Controller (not a @RestController) so it can handle the redirect. The last thing we need is a mapping from an unauthenticated response (HTTP 401, a.k.a. UNAUTHORIZED) to the "/unauthenticated" endpoint we just added:
``` java
@Configuration
public class ServletCustomizer {
  @Bean
  public EmbeddedServletContainerCustomizer customizer() {
    return container -> {
      container.addErrorPages(
          new ErrorPage(HttpStatus.UNAUTHORIZED, "/unauthenticated"));
    };
  }
}
```

4、Generating a 401 in the Server
A 401 response will already be coming from Spring Security if the user cannot or does not want to login with Github, so the app is already working if you fail to authenticate (e.g. by rejecting the token grant).

To spice things up a bit we will extend the authentication rule to reject users that are not in the right organization. It is easy to use the Github API to find out more about the user, so we just need to plug that into the right part of the authentication process. Fortunately, for such a simple use case, Spring Boot has provided an easy extension point: if we declare a @Bean of type **AuthoritiesExtractor** it will be used to construct the authorities (typically "roles") of an authenticated user. We can use that hook to assert the the user is in the correct orignization, and throw an exception if not:  

```java
@Bean
public AuthoritiesExtractor authoritiesExtractor(OAuth2RestOperations template) {
  return map -> {
    String url = (String) map.get("organizations_url");
    @SuppressWarnings("unchecked")
    List<Map<String, Object>> orgs = template.getForObject(url, List.class);
    if (orgs.stream()
        .anyMatch(org -> "spring-projects".equals(org.get("login")))) {
      return AuthorityUtils.commaSeparatedStringToAuthorityList("ROLE_USER");
    }
    throw new BadCredentialsException("Not in Spring Projects origanization");
  };
}
```
Note that we have autowired a OAuth2RestOperations into this method, so we can use that to access the Github API on behalf of the authenticated user. We do that, and loop over the organizations, looking for one that matches "spring-projects" (this is the organization that is used to store Spring open source projects). You can substitute your own value there if you want to be able to authenticate successfully and you are not in the Spring Engineering team. If there is no match, we throw BadCredentialsException and this is picked up by Spring Security and turned in to a 401 response.

The OAuth2RestOperations has to be created as a bean as well (as of Spring Boot 1.4), but that’s trivial because its ingredients are all autowirable by virtue of having used @EnableOAuth2Sso:
```java
@Bean
public OAuth2RestTemplate oauth2RestTemplate(OAuth2ProtectedResourceDetails resource, OAuth2ClientContext context) {
	return new OAuth2RestTemplate(resource, context);
}
```

Obviously the code above can be generalized to other authentication rules, some applicable to Github and some to other OAuth2 providers. All you need is the OAuth2RestOperations and some knowledge of the provider’s API.


