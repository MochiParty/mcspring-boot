# Spring Boot Spigot Starter

[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/Alan-Gomes/mcspring-boot/fork)
[![Maven Central](https://img.shields.io/maven-central/v/dev.alangomes/spigot-spring-boot-starter.svg)](https://search.maven.org/#artifactdetails%7Cdev.alangomes%7Cspigot-spring-boot-starter%7C0.12.0%7Cjar)
[![License](https://img.shields.io/github/license/Alan-Gomes/mcspring-boot.svg?style=popout)](https://github.com/Alan-Gomes/mcspring-boot/blob/master/LICENSE)
[![Coverage Status](https://img.shields.io/coveralls/github/Alan-Gomes/mcspring-boot/master.svg)](https://coveralls.io/github/Alan-Gomes/mcspring-boot?branch=master)
[![GitHub Issues](https://img.shields.io/github/issues/Alan-Gomes/mcspring-boot.svg)](https://github.com/Alan-Gomes/mcspring-boot/issues)
[![CircleCI Status](https://img.shields.io/circleci/project/github/Alan-Gomes/mcspring-boot/master.svg)](https://circleci.com/gh/Alan-Gomes/mcspring-boot)

> A Spring boot starter for Bukkit/Spigot/PaperSpigot plugins

## Features

- Easy setup
- Full [Picocli](http://picocli.info/) `@Command` support (thanks to [picocli-spring-boot-starter](https://github.com/kakawait/picocli-spring-boot-starter)) 
- Secure calls with `@Authorize`
- Automatic `Listener` registration
- Session system
- Designed for testability
- Full Spring's dependency injection support
- Easier Bukkit main thread synchronization via `@Synchronize`
- Support Spring scheduler on the bukkit main thread (`@Scheduled`)

## Getting started

Add the Spring boot starter to your project

```xml
<dependency>
  <groupId>dev.alangomes</groupId>
  <artifactId>spigot-spring-boot-starter</artifactId>
  <version>0.12.0</version>
</dependency>
```

Create a class to configure the Spring boot application

```java
@SpringBootApplication(scanBasePackages = "me.test.testplugin")
public class Application {

}
```

Then create the plugin main class using the standard spring initialization, just adding the `SpringSpigotInitializer` initializer.  

<a name="initialization"></a> 
```java
public class ExamplePlugin extends JavaPlugin {

    private ConfigurableApplicationContext context;

    @Override
    public void onEnable() {
        saveDefaultConfig();
        ResourceLoader loader = new DefaultResourceLoader(getClassLoader());
        SpringApplication application = new SpringApplication(loader, Application.class);
        application.addInitializers(new SpringSpigotInitializer(this));
        context = application.run();
    }

    @Override
    public void onDisable() {
        context.close();
        context = null;
    }

}
```

And that's it! Your plugin is ready to use all features from Picocli and Spring Boot!

> **Note**: If you are getting a `No auto configuration classes found in META-INF/spring.factories` error, it means
that your build configuration must be wrong, see [this issue](https://github.com/Alan-Gomes/mcspring-boot/issues/2#issuecomment-486331102) for more details.

## Creating a simple command

All commands are based in the [Picocli's API](http://picocli.info/), the only difference is that the classes are
automatically registered at the start of the plugin.
To allow auto registration, you also need to annotate all classes with `@Component`.

Example of a command:

```java
@Component
@CommandLine.Command(name = "hello")
public class HelloCommand implements Callable<String> {

    @CommandLine.Parameters(index = "0", defaultValue = "world")
    private String name;

    @Override
    public String call() {
        return "hello " + name;
    }
}
```

If you need the sender of the command, you can also retrieve it in the `Context`:

```java
@Component
@CommandLine.Command(name = "heal")
public class HealCommand implements Runnable {

    @Autowired
    private Context context;

    @Override
    public void run() {
        Player player = context.getPlayer();
        player.setHealth(20);
    }
}
```

In addition, you can also inject the `Plugin` and `Server` instances via `@Autowired`

### Working with subcommands

Sometimes your command gets too complex and need to be splitted in subcommands. The Picocli API already supports
subcommands using two different ways:

The first and easiest way is to create subcommands with nested classes:

```java
@Component
@CommandLine.Command(name = "money")
public class MoneyCommand {

    @Component
    @CommandLine.Command(name = "add")
    public class AddCommand implements Runnable {

        @Autowired
        private Context context;

        @Override
        public void run() {
            context.getSender().sendMessage("test");
        }
    }
}
```

The second and most *clean* way is to use *annotated subcommands*, which allows you to create each subcommand in a completely
separate class. To do this, you need to create the subcommand class with the `@Subcommand` annotation instead:

```java
@Subcommand
@CommandLine.Command(name = "add")
public class MoneyAddCommand implements Runnable {

    @Autowired
    private Context context;

    @Override
    public void run() {
        context.getSender().sendMessage("test");
    }
}
```

After that, you can add a reference to the subcommand inside the **root** command, just like this:

```java
@Component
@CommandLine.Command(
        name = "money",
        subcommands = {MoneyAddCommand.class}
)
public class MoneyCommand {

    // some code
    
}
```

## Retrieving configuration

Is really easy to retrieve configuration properties, you can use the `@DynamicValue` annotation, it will automatically
lookup your `config.yml` to find the value, otherwise will fallback to the framework's properties.

Example:

```java
@DynamicValue("${command.delay}")
private Instance<Integer> commandDelay;
```

You can also use Spring built-in `@Value` annotation, which works the same way, but doesn't support configuration reloading. 

## Securing methods

> "I love writing authentication and authorization code." ~ No Developer Ever.

Creating authorization/permission checking code is really boring, to make it easier, this starter implements the `@Authorize`
annotation, which allows you to define rules to prevent some method to be ran by some players.

The annotation expects a expression in [Spring Expression Language (SpEL)](https://docs.spring.io/spring/docs/3.0.x/reference/expressions.html#expressions-language-ref)
which will be evaluated over the player in the current [context](#context). If the expression evaluates to `false` or there's
no player in the context, the method will automatically throw a `PermissionDeniedException` and `PlayerNotFoundException`, respectively.

Example:

```java
@Authorize("hasPermission('myplugin.dosomething') or getHealth() > 10")
public void doSomething() {
    
}
```

**WARNING**: Due to a limitation in the [Picocli](http://picocli.info/) reflection implementation, the use of `@Authorize`
 as well as `@Synchronize` and `@Audit` is not supported on command classes, it is highly recommended to make use of these 
 features in a separate class, for example: a service.
 
 ## Scheduling
 
 Creating scheduled tasks is easier than ever, you can simply use the `Scheduler` bean, which allows you to create the tasks
 without needing to have a reference to the `Plugin` or `Server`. This custom scheduler also keeps the current context
 during the execution(s), everything just works.
 
 Example:
 
 ```java
 @Service
 class AnnounceService {
    
     @Autowired
     private Scheduler scheduler;
     
     @Autowired
     private Context context;
     
     @Autowired
     private ChatService chatService;
 
     public void announce(String message, long delay) {
         scheduler.scheduleSyncDelayedTask(() -> {
             chatService.broadcast(context.getPlayer().getName() + " said " + message);
         }, delay);
     }
 
 }
 ```
 
 ## Sessions
 
 Using the power of [contexts](#context), we introduce the concept of *sessions*: a per user key-value storage, which
 allows you to work using the same idea of web session. Each session lives during the player time on the server, and is
 cleared after logout.
 
The usage is pretty simple, you just need to inject a `SessionService` and done. All methods are based in a map, which
is bound to the player context, this means that the user session will be automatically managed, no worries. 

## <a name="context"></a> Understanding contexts

Different from a regular web application, a Bukkit server does not have a concept of session,
all command and event executions are (almost) in the same thread without any definition of the sender in context
(aka who triggered the event/command).

To circumvent this limitation and allow the identification of the sender, this starter implements the `Context`,
a bean that stores senders based on the current thread id. Since almost every execution is in the same thread
(the main server thread), it is safe to store a single user per time, if the execution is asynchronous, it will use another context.

To understand this better, let's take a look on this example:

```java
@Service
class BusinessService {
    
    @Autowired
    private Context context;
    
    @Autowired
    private ChatService chatService;

    public void sudoHello(Player player) {
        context.runWithSender(player, () -> {
            chatService.sayHello();
        });
        // or, if you are already familiar with Java 8 method references
        context.runWithSender(player, chatService::sayHello);
    }

}
```

As you can see, we override the sender (player) of the current context.
With this setting, the `ChatService` and all dependent services will see the `player` as the current player in the context, also,
every method containing `@Authorize` or `@Audit` will be able to detect the this player to apply the authorization
rules or to display in the log.

**Note**: All commands and events are automatically intercepted to set the player in the context, the `runWithSender` method exists
just to handle special cases.

## Synchronization

In a regular Bukkit plugin, you don't need to care about threads and synchronization. But if you want to run something
outside the server main thread, like in `BukkitScheduler#runTaskAsynchronously​()`, every access to Bukkit API needs to be
synchronized to make sure you don't get a `IllegalStateException`.

To make this synchronization easier (without using schedulers and creating runnables), you can simply use the `@Synchronize`
annotation, which automatically schedules all calls under the hood if they are not in the main thread, passing through otherwise.

**Important things if the call get scheduled**:
- The return value of the method will **always** be `null`, so make sure your code is null-safe.
- The method will **not** run immediately, it will on the next server tick, so make sure your code don't rely on that.

## Testing

This starter is designed to allow writing tests as easy as possible. Since Bukkit's own design is not the best for testing, it's
still necessary to make some mocks in the server classes.

First, you need to implement some mocks for the `Player`, `Server`, `FileConfiguration` and `BukkitScheduler` classes.
Don't worry, the basic implementation is already done and should fit for most test cases, you just need to copy [this class](https://github.com/Alan-Gomes/mcspring-boot/blob/master/src/test/java/dev/alangomes/test/util/SpringSpigotTestInitializer.java) into your project test files.

After setting up the mocks, you can start writing tests using this structure:

```java
@RunWith(SpringRunner.class)
@ContextConfiguration(
        classes = MyApplication.class,
        initializers = SpringSpigotTestInitializer.class
)
@DirtiesContext(classMode = ClassMode.AFTER_EACH_TEST_METHOD)
public class SomeServiceTest {
    
    @Autowired
    private SomeService someService;
    
    // your tests here
    
}
```

Like any traditional spring tests, you can inject and mock beans, as well as use any starters for testing. 