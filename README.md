# Spring-Security-deep-dive


## Intro
When an HTTP request reaches a Spring Security enabled application, it doesn’t go straight to your controller. Instead, it first enters a chain of security filters that handle authentication, authorization, and context persistence.

Understanding this flow is essential if you want to customize security or debug why certain authentication mechanisms aren’t working as expected. Let’s break it down step by step.


## How a Request Enters Spring Security?

When a request first hits your application, it goes through the **Servlet Container’s filter chain**. This chain is made of filters implementing `javax.servlet.Filter`.
Among these filters, one of the most important is **`DelegatingFilterProxy`**.


## What is `DelegatingFilterProxy`?

The `DelegatingFilterProxy` acts as a **bridge** between the **Servlet Container lifecycle** and the **Spring ApplicationContext**.
The **Servlet Container** (e.g., Tomcat) manages a set of filters, but it has no knowledge of Spring-managed beans.
On the other hand, Spring Security registers its filters as beans inside the **ApplicationContext**.
Without a bridge, the container’s filters and Spring’s beans would live in two separate worlds.

### How it Works
 The Servlet Container delegates the request to `DelegatingFilterProxy`.
 The `DelegatingFilterProxy` looks inside the Spring `ApplicationContext` for a bean named **`springSecurityFilterChain`**.
 This bean is an instance of **`FilterChainProxy`**, which itself implements `javax.servlet.Filter`
 Once found, the proxy simply delegates the request to `FilterChainProxy`.
 At this point, the `DelegatingFilterProxy`’s job is done. The heavy lifting is now handled by `FilterChainProxy`.




## `FilterChainProxy` and `SecurityFilterChain`

**FilterChainProxy** is a special Filter provided by Spring Security that allows delegating to many Filter instances through SecurityFilterChain. Since **FilterChainProxy** is a Bean, it is typically wrapped in a **DelegatingFilterProxy** (**springSecurityFilterChain** bean).

Every incoming request in a Spring Security application first passes through the **`FilterChainProxy`**.
The `FilterChainProxy` is responsible for deciding which `SecurityFilterChain` should be executed for a given `HttpServletRequest`.

- You can have multiple `SecurityFilterChain`s in your configuration.
- `FilterChainProxy` chooses the appropriate one based on the incoming request.
- Once selected, it executes all filters defined within that `SecurityFilterChain`.


The **first filter** that runs inside this chain is usually the `SecurityContextHolderFilter`.

## `SecurityContextHolderFilter`

When working with Spring Security, one of the first filters that gets executed in the filter chain is the **`SecurityContextHolderFilter`**. This filter is responsible for preparing and binding the `SecurityContext` to the current request lifecycle.

#### How `SecurityContextHolderFilter` works?

 It checks if the filter was already applied for the current request to avoid duplicate execution.
 ```java
 if (request.getAttribute(FILTER_APPLIED) != null) {    
        chain.doFilter(request, response);  
    }
```
 
 If not applied, it marks the request as processed and then calls `securityContextRepository`:
```java
Supplier<SecurityContext> deferredContext = this.securityContextRepository.loadDeferredContext(request);

```

Then add the deferred context to `SecurityContextHolderStrategy`(explained later in this article).
```java
try {  
    this.securityContextHolderStrategy.setDeferredContext(deferredContext);  
    chain.doFilter(request, response);  
} finally {  
    this.securityContextHolderStrategy.clearContext();  
    request.removeAttribute(FILTER_APPLIED);  
}
```



---
**`SecurityContextHolderFilter`** Class
```java

private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws ServletException, IOException { 
    //checks if the filter was already applied for the current request
    if (request.getAttribute(FILTER_APPLIED) != null) {    
        chain.doFilter(request, response);  
    } else {  
        request.setAttribute(FILTER_APPLIED, Boolean.TRUE);  
        Supplier<SecurityContext> deferredContext = this.securityContextRepository.loadDeferredContext(request);  
  
        try {  
            this.securityContextHolderStrategy.setDeferredContext(deferredContext);  
            chain.doFilter(request, response);  
        } finally {  
            this.securityContextHolderStrategy.clearContext();  
            request.removeAttribute(FILTER_APPLIED);  
        }  
  
    }  
}


```


### What is`SecurityContextRepository`

`SecurityContextRepository` is an interface responsible for loading and storing the `SecurityContext`. It defines two important methods:
1. `loadContext`
2. `loadDeferredContext` (a default method that internally uses `loadContext`) used to lazy loading `securityContext`


**`SecurityContextRepository`** interface  has five implementations.

1. **`DelegatingSecurityContextRepository`**: Delegates loading/saving of the `SecurityContext` to multiple repositories (a composite wrapper).

2. **`HttpSessionSecurityContextRepository`**: Stores/Retrieves the `SecurityContext` in the user’s HTTP session (default in Spring Security).

3. **`NullSecurityContextRepository`**: Does nothing (never stores or retrieves a `SecurityContext`)  useful for stateless apps (like pure JWT).

4. **`RequestAttributeSecurityContextRepository`**: Stores the `SecurityContext` only within the current request lifecycle.

5. **`TestSecurityContextRepository`**: A repository used in tests (Spring Security’s test support) to supply a `SecurityContext` in mock requests.


### `DelegatingSecurityContextRepository`

By default, Spring Security uses the **DelegatingSecurityContextRepository** when the application is configured with session-based (Stateful) authentication.
`DelegatingSecurityContextRepository` allows Spring Security to **delegate the responsibility of loading, saving, and checking the `SecurityContext` to multiple `SecurityContextRepository` implementations** instead of just one.

Spring Security wires in a `DelegatingSecurityContextRepository`, which delegates to two other repositories by default:
- `HttpSessionSecurityContextRepository`
- `RequestAttributeSecurityContextRepositor`

#### How it works?

You give it a list of repositories (like `HttpSessionSecurityContextRepository`, `RequestAttributeSecurityContextRepository`, etc.),  
It tries them **one by one**, deciding which one to use.
#### `DelegatingSecurityContextRepository.loadDeferredContext()` 

The default `DelegatingSecurityContextRepository` uses `loadDeferredContext(HttpServletRequest)` instead of eagerly calling `loadContext()`

Here’s the relevant code:
```java
public DeferredSecurityContext loadDeferredContext(HttpServletRequest request) {
    DeferredSecurityContext deferredSecurityContext = null;

    for (SecurityContextRepository delegate : this.delegates) {
        if (deferredSecurityContext == null) {
            deferredSecurityContext = delegate.loadDeferredContext(request);
        } else {
            DeferredSecurityContext next = delegate.loadDeferredContext(request);
            deferredSecurityContext =
                    new DelegatingDeferredSecurityContext(deferredSecurityContext, next);
        }
    }

    return deferredSecurityContext;
}

```


**Explain the above code**
Start with `deferredSecurityContext = null`.
The `DeferredSecurityContext` interface is implemented mainly by two classes
**`SupplierDeferredSecurityContext`**, which lazily loads the context using a `Supplier`, and  
**`DelegatingDeferredSecurityContext`**, which chains multiple deferred contexts together.

Instead of eagerly loading the `SecurityContext`, Spring stores a `DeferredSecurityContext` within the request. The actual `SecurityContext` is loaded lazily only when `.get()` is invoked for the first time.

**Iterating over repositories**
For the first `SecurityContextRepository`, call `loadDeferredContext()` and set it as the base.
For each additional repository, call `loadDeferredContext()` again and wrap it together with the existing one using` DelegatingDeferredSecurityContext`.

**Chaining with `DelegatingDeferredSecurityContext`**
The inner class `DelegatingDeferredSecurityContext` combines two contexts.
When you call .get() on it, it first checks the “previous” context.
If that context has a valid Authentication (using isGenerated()), it returns it.
Otherwise, it delegates to the “next” context.

`DelegatingDeferredSecurityContext` code:
```java
static final class DelegatingDeferredSecurityContext implements DeferredSecurityContext {  
    private final DeferredSecurityContext previous;  
    private final DeferredSecurityContext next;  
  
    DelegatingDeferredSecurityContext(DeferredSecurityContext previous, DeferredSecurityContext next) {  
        this.previous = previous;  
        this.next = next;  
    }  
  
    public SecurityContext get() {  
        SecurityContext securityContext = (SecurityContext)this.previous.get();  
        return !this.previous.isGenerated() ? securityContext : (SecurityContext)this.next.get();  
    }  
  
    public boolean isGenerated() {  
        return this.previous.isGenerated() && this.next.isGenerated();  
    }  
}
```


### How `DelegatingDeferredSecurityContext` Handles Multiple Repositories

In the code we looked at earlier, you might notice that `DelegatingDeferredSecurityContext` only stores **two fields**: `previous` and `next`.  
So the natural question is:

> _What happens if we configure more than two `SecurityContextRepository` instances in the delegates list?_


#### Step-by-Step Construction

Let’s imagine you have three repositories: **R1, R2, R3**.
1. The first one (`R1`) becomes the initial `DeferredSecurityContext` (Actually `SupplierDeferredSecurityContext`).
2. When the loop processes `R2`, Spring creates:

```java
deferredSecurityContext =
    new DelegatingDeferredSecurityContext(R1, R2);

```

3. When it reaches `R3`, it wraps the chain again:
```java
deferredSecurityContext =
    new DelegatingDeferredSecurityContext(
        new DelegatingDeferredSecurityContext(R1, R2),
        R3
    );

```

When you call `.get()` on this final chain:
1. It first tries `previous` (which itself may be another `DelegatingDeferredSecurityContext`) That in turn checks `R1`, and if not valid, `R2`.
2. If neither has a valid `Authentication`, it moves on to `next` (`R3`).

This chaining continues for however many repositories you configure.

---


now we saw that **`DelegatingSecurityContextRepository`** doesn’t handle persistence directly.  
Instead, it **delegates** to a list of repositories that you add or use the defualt.
By default, this list contains:

`HttpSessionSecurityContextRepository`:  Stores/Retrieves the `SecurityContext` in the user’s HTTP session (default in Spring Security)

`RequestAttributeSecurityContextRepository`: Stores the `SecurityContext` only in the current request attributes (lifetime = single request).

whenever Spring Security needs to load the context, it loops through these repositories and calls their `loadDeferredContext()` methods.
`HttpSessionSecurityContextRepository.loadDeferredContext()`
`RequestAttributeSecurityContextRepository.loadDeferredContext()`



###  `SupplierDeferredSecurityContext`


>*Quick Recap: What is a `Supplier`?*
>*In Java, a `Supplier<T>` is a **functional interface** that returns a value of type `T` when you call*
>*`get()`.*
>*is often used for lazy evaluation or deferred execution, where the actual result is only generated when needed.*
>*it can be implemented using lambda expressions or method references*

```java
@FunctionalInterface  
public interface Supplier<T> {  
    
      T get();  
}
```

How `SupplierDeferredSecurityContext` works:

- When you call `securityContextRepository.loadDeferredContext(request)`, it doesn’t give you the `SecurityContext` directly.
- Instead, it wraps a **`Supplier<SecurityContext>`** inside a `SupplierDeferredSecurityContext`.
- the `Supplier<SecurityContext>` is used to **lazily provide the `SecurityContext`** only when `get()` is called. This avoids unnecessary work if the context is never accessed in the request lifecycle(Example: public requests)
```java
default DeferredSecurityContext loadDeferredContext(HttpServletRequest request) {  
    Supplier<SecurityContext> supplier = () -> this.loadContext(new HttpRequestResponseHolder(request, (HttpServletResponse)null));  
    return new SupplierDeferredSecurityContext(SingletonSupplier.of(supplier), SecurityContextHolder.getContextHolderStrategy());  
}
```



---

After we’ve already walked through how `securityContextRepository.loadDeferredContext(request)` works end-to-end now let’s move to the next step.

the  returned `deferredSecurityContext` will be passed into  
`securityContextHolderStrategy.setDeferredContext(deferredContext);` (See `SecurityContextHolderFilter`).
This step binds the deferred context to `SecurityContextHolderStrategy`.  
By default, this strategy is `ThreadLocalSecurityContextHolderStrategy`,  
which means the `SecurityContext` will be stored in a `ThreadLocal` variable associated with the current request thread .
This allows the `SecurityContext` to be accessible from anywhere in the application during the request lifecycle.

This step is important because it makes the `securityContext` available through the `SecurityContextHolder` is where Spring Security stores the details of who is authenticated. So, whenever we later call `SecurityContextHolder.getContext()`, it will actually retrieve the context from this deferred strategy.



### `BasicAuthenticationFilter`

Basic Authentication is a straightforward method for securing APIs by requiring a username and password for each request.
The `BasicAuthenticationFilter` Processes a HTTP request's BASIC authorization headers, putting the result into the `SecurityContextHolder`.

>*Note:* 
>*the filter sets the context through the SecurityContextHolderStrategy (not directly on `SecurityContextHolder`) as we will explain.*


#### Extracting Credentials - `AuthenticationConverter`

The filter begins by using an `AuthenticationConverter`,  
typically a `BasicAuthenticationConverter`, which looks for the following header:
```http
Authorization: Basic base64(username:password)
```
If no valid header exists, the filter skips authentication and continues the chain.


Then The converter decodes this header and creates an **unauthenticated**  
`UsernamePasswordAuthenticationToken`.
```java
byte[] base64Token = header.substring(6).getBytes(StandardCharsets.UTF_8);  
byte[] decoded = this.decode(base64Token);
```

```java
UsernamePasswordAuthenticationToken result = UsernamePasswordAuthenticationToken.unauthenticated(token.substring(0, delim), token.substring(delim + 1));
```


Finally, the converter enriches the token with contextual request details
such as the client’s IP address and session ID by attaching a `WebAuthenticationDetails` instance:
```java
result.setDetails(this.authenticationDetailsSource.buildDetails(request));

```

At this point, the converter returns an authentication token that holds both the extracted credentials and request metadata,  
ready to be passed to the `AuthenticationManager` for verification.


#### Authenticating the Request - `AuthenticationManager` (`ProviderManager`)

If credentials(`UsernamePasswordAuthenticationToken` not null) exist, the filter delegates to the configured **`AuthenticationManager`**:
```java
Authentication authResult = this.authenticationManager.authenticate(authRequest);
```
Typically, the `AuthenticationManager` is a `ProviderManager`,  
which iterates through a list of `AuthenticationProviders` and check which one suitable most commonly, the `DaoAuthenticationProvider`.
That provider finds the user with a user details service and validates the password using a password encoder.


Once authentication succeeds, `BasicAuthenticationFilter ` creates a new empty `SecurityContext` using the configured `SecurityContextHolderStrategy` and stores the authenticated `Authentication` inside it:
```java
SecurityContext context = this.securityContextHolderStrategy.createEmptyContext();  
context.setAuthentication(authResult);  
this.securityContextHolderStrategy.setContext(context);
```

This allows the `SecurityContext` to be accessible from anywhere in the application during the request lifecycle.
now you can access the SecurityContext by `SecurityContextHolder.getContext()`

Next, the Filter saves the context using a `SecurityContextRepository` like  `HttpSessionSecurityContextRepository` to be available in the user’s HTTP session (default in Spring Security)
```java
this.securityContextRepository.saveContext(context, request, response);

```


Once the user is authenticated, the filter calls:
```java
rememberMeServices.loginSuccess(request, response, authentication)`
````
Next, `rememberMeServices.loginSuccess(request, response, authentication)` is invoked.  
This allows Spring Security’s `RememberMeServices` implementation to create and send a new _remember-me cookie_ to the client after successful authentication.


**Remember-me** or persistent-login authentication refers to web sites being able to remember the identity of a principal between sessions. This is typically accomplished by sending a cookie to the browser, with the cookie being detected during future sessions and causing automated login to take place. Spring Security provides the necessary hooks for these operations to take place and has two concrete remember-me implementations. One uses hashing to preserve the security of cookie-based tokens and the other uses a database or other persistent storage mechanism to store the generated tokens.

Spring Security provides two implementation for Remember-Me:

- Simple Hash-Based Token Approach: use hashing to preserve the security of cookie-based tokens.
- Persistent Token Approach: use a database or other persistent storage mechanism to store the generated tokens.


Remember-me is used with `UsernamePasswordAuthenticationFilter` and is implemented through hooks in the `AbstractAuthenticationProcessingFilter` superclass. It is also used within `BasicAuthenticationFilter`. The hooks invoke a concrete `RememberMeServices` at the appropriate times. The following listing shows the interface:
```java

Authentication autoLogin(HttpServletRequest request, HttpServletResponse response);

void loginFail(HttpServletRequest request, HttpServletResponse response);

void loginSuccess(HttpServletRequest request, HttpServletResponse response,
	Authentication successfulAuthentication);
```

The Filter calls only the `loginFail()` and `loginSuccess()` methods. The `autoLogin()` method is called by `RememberMeAuthenticationFilter` whenever the `SecurityContextHolder` does not contain an `Authentication`.

**`loginFail`**: Called whenever an interactive authentication attempt was made, but the credentials supplied by the user were missing or otherwise invalid.
**`loginSuccess`**: Called whenever an interactive authentication attempt is successful.




 **If authentication fails, then _Failure_.**

1. The  SecurityContextHolder is cleared out (`SecurityContextHolderStrategy`).
2. `RememberMeServices.loginFail` is invoked. If remember me is not configured, this is a no-op. See the `RememberMeServices`  interface in the Javadoc.
3. `AuthenticationEntryPoint` is invoked to trigger the WWW-Authenticate to be sent again. See the `AuthenticationEntryPoint` interface in the Javadoc.


 **If authentication is successful, then _Success_.**

1. The Authentication is set on the SecurityContextHolder (`SecurityContextHolderStrategy`).
2. `RememberMeServices.loginSuccess` is invoked. If remember me is not configured, this is a no-op. See the `RememberMeServices` interface in the Javadoc.
3. The `BasicAuthenticationFilter` invokes `FilterChain.doFilter(request,response)` to continue with the rest of the application logic. See the `BasicAuthenticationFilter` Class in the Javadoc




