First, create an app:

```java
public final class App {
  public static void main(final String... args) {
    new Server(
      new PgsRegex()
        .with("/robots.txt", "")
        .with("/", new PgIndex())
        .with("/xsl/.*", new PgClasspath())
        .with(
          "/account", 
          new Page.Source() {
            @Override
            public Page page(final Request request) {
              return new PgAccount(users, request);
            }
          }
        )
        .with(
          "/balance/(?<user>[a-z]+)", 
          new PgsRegex.Source() {
            @Override
            public Page page(final RqRegex request) {
              return new PgBalance(request);
            }
          }
        )
    ).listen();
  }
}
```

Here is a simple page:

```java
@Immutable
public final class PgIndex implements Page {
  @Override
  public Response draw() {
    return new RsHtml("<html>Hello, world!</html>");
  }
}
```

Let's create a more complex page:

```java
@Immutable
public final class PgAccount implements Page {
  private final User user;
  public PgAccount(final Users users, final Request request) {
    this.user = users.find(new RqCookies(request).get("user"));
  }
  @Override
  public Response draw() {
    return new RsLogin(
      new RsXSLT(
        new RsXembly(
          new XePlain("PI 'xsl-stylesheet', 'href=\"/xsl/account.xsl\"'"),
          new XePlain("ADD 'page'"),
          new XePrepend("XPATH '/page'", this.user)
        )
      ),
      this.user
    );
  }
}
```

Now let's see how that `User` class should look like:

```java
@Immutable
public final class User implements XeSource {
  private final String name;
  private final int balance;
  @Override
  public Iterable<Directive> toXembly() {
    return new Directives().add("user")
      .add("name").set(this.name).up()
      .add("balance").set(Integer.toString(this.balance));
  }
  @Override
  public String toString() {
    return this.name;
  }
}
```

Here is how `RsLogin` may look like:

```java
@Immutable
public final class RsLogin extends Response.Wrap {
  public RsLogin(final Response response, final User user) {
    super(
      new RsCookied(response).with(
        "user", user.toString()
      )
    );
  }
}
```

Here is how we can deal with JSON:

```java
@Immutable
public final class PgBalance extends Page.Fixed {
  public PgBalance(final RqRegex request) {
    super(new RsJSON(request.matcher().group("user"))));
  }
}
```

This is the method to add to `User`:

```java
@Immutable
public final class User implements XeSource, RsJSON.Source {
  @Override
  public JsonObject toJSON() {
    return Json.createObjectBuilder()
      .add("balance", this.balance)
      .build();
  }
}
```

Here is how key interfaces look like. First, the `Request`:

```java
@Immutable
public interface Request {
  String method();
  String uri();
  Iterable<String> headers();
  InputStream body();
}
```

And the `Response`:

```java
@Immutable
public interface Response {
  int status();
  String line();
  Iterable<String> headers();
  InputStream body();
}
```

## RsXembly

Here is how you generate an XML page using Xembly:

```java
Response response = new RsXembly(
  new XePlain("ADD 'page'"),
  new XePrepend("XPATH '/page'", this.user)
)
```

This is a complete example, with all possible options:

```java
Response response = new RsXembly(
  new XeRoot("page"),
  new XeMillis(false),
  new XeStylesheet("/xsl/account.xsl"),
  new XeToRoot(this.user),
  new XeSource() {
    @Override
    public Iterable<Directive> toXembly() {
      return new Directives().add("status").set("alive");
    }
  },
  new XeMillis(true),
)
```

This is the output that will be produced:

```xml
<?xml version='1.0'?>
<?xsl-stylesheet href='/xsl/account.xsl'?>
<page millis='5664'>
  <user>
    <name>Jeff Lebowski</name>
    <balance>123</balance>
  </user>
  <status>alive</status>
</page>
```

