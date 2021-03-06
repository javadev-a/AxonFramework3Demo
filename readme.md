#Axon Framework 3.0 with Spring Boot
___
## About this Project

This is my first attempt at using Axon Framework version 3. Due to some differences between the Milestone releases and the Full Release, I had to modify some of the code. You will see the differences in the **[Differences](#differences)** section in this Readme.
**(December 30th 2016) Version 3.0 had just come out so it makes sense to use the latest.**

I followed these three videos and added my own spin as I wanted:

 - [Getting started with Axon 3 Live coding 1](https://www.youtube.com/watch?v=s2zH7BsqtAk).
 
 - [Getting started with Axon 3 Live coding 2](https://www.youtube.com/watch?v=Fj365BufWNU).
 
 - [Getting started with Axon 3 Live coding 3](https://www.youtube.com/watch?v=qqk2Df_0Pm8).

###Credit - Getting started with Axon 3 Live coding sessions 1-3
**by**: [Trifork Webinar Channel](https://www.youtube.com/channel/UCz9eNSe8kY7z8DEyvv-slZg)

 
 At the time of writing this project the documentation was incomplete. Check [here](https://docs.axonframework.org/v/3.0/) for the latest documentation. 
 
___
 
## Maven:
 
 In my maven pom I added these four dependencies rather than the milestones:
 
 ```XML
 ...
<dependency>
	<groupId>org.axonframework</groupId>
	<artifactId>axon-core</artifactId>
	<version>3.0</version>
</dependency>
<dependency>
	<groupId>org.axonframework</groupId>
	<artifactId>axon-test</artifactId>
	<version>3.0</version>
</dependency>
<dependency>
	<groupId>org.axonframework</groupId>
	<artifactId>axon-spring-boot-autoconfigure</artifactId>
	<version>3.0</version>
</dependency>
<!-- You will need this in video 2-->
<dependency>
	<groupId>javax.inject</groupId>
	<artifactId>javax.inject</artifactId>
	<version>1</version>
</dependency>
...
 ```


___

## Differences
**-What I had to change to make things work**

###Video 1
- Use H2 instead of hsqldb. This will allow you to easily see what is happening in the H2 db via the web console. Take a look at the `application.yml`.
- The way you instantiate an Aggregate fixture is not:
``` Java
...
    private FixtureConfiguration<Account> fixture;

    @Before
    public void setUp() throws Exception {
        fixture = Fixtures.newGivenWhenThenFixture(Account.class);
    }
...
```
And is
``` Java
...
import org.axonframework.test.aggregate.AggregateTestFixture;
import org.axonframework.test.aggregate.FixtureConfiguration;
...

    private FixtureConfiguration<Account> fixture;

    @Before
    public void setUp() throws Exception {
        fixture = new AggregateTestFixture(Account.class);
    }
```
- You no longer need to use the annotation @EnableAxonAutoConfiguration on your Spring Boot Application class. The  `axon-spring-boot-autoconfigure` dependency now takes care of that.
- When testing instead of placing the "test code" in the main class, I  create a proper Integration test and check that the context runs:
```Java
    @Test
    public void when_startingUpTheApplicationContext_expect_CommandBussuccessful() throws SQLException {
        CommandBus commandBus = context.getBean(CommandBus.class);
        assertNotNull("Application context not initialised successfully", context);
        assertNotNull("Application CommandBus not initialised successfully", commandBus);

        commandBus.dispatch(asCommandMessage(new CreateAccountCommand("54321", 500)));
        commandBus.dispatch(asCommandMessage(new WithdrawMoneyCommand("54321", 250)));
    }
```
- I do not implement the AsynchronousCommandBus because at this point we don't need it. However here is the bean code you will need to add to your config:
```Java
    @Bean
    public CommandBus getCommandBus() {
        return new AsynchronousCommandBus();
    }
```

###Video 2

- The way you instantiate a Saga fixture is not:
``` Java
...
    private FixtureConfiguration fixture;
    
    @Before
    public void setUp() {
        fixture = new AnnotatedSagaTestFixture<>(MoneyTransfer.class);
    }
...
```
And is
``` Java
...
import org.axonframework.test.saga.FixtureConfiguration;
import org.axonframework.test.saga.SagaTestFixture;
...

    private FixtureConfiguration fixture;

    @Before
    public void setUp() {
        fixture = new SagaTestFixture<>(MoneyTransfer.class);
    }
...
```
- When you test a saga, there is a subtle difference when you check the output of your dispatched commands. This is because the full release now has a `Matchers` Class. Therefore for ease of use just use `expectDispatchedCommands(...)` method:
``` Java
...
    fixture.givenNoPriorActivity()
         .whenPublishingA(new RequestedMoneyTransferEvent("tf1","acc1t","acct2",100))
         .expectActiveSagas(1)
         .expectDispatchedCommandsEqualTo(WithdrawMoneyCommand("acct1",100));
...
```
Will need to be:
``` Java
...
    fixture.givenNoPriorActivity()
         .whenPublishingA(new RequestedMoneyTransferEvent("tf1","acct1","acct2",100))
         .expectActiveSagas(1)
         .expectDispatchedCommands(new WithdrawMoneyCommand("acct1", 100));
...
```
- Some where between the two webinars there is code missing from Account:
``` Java
    ...
    @CommandHandler
    public void handle(DepositMoneyCommand command) {
        apply(new MoneyWithdrawnEvent(this.accountId, command.getTransactionId(), command.getAmount(), balance + command.getAmount()));
    }
    
    @EventSourcingHandler
    public void on(MoneyWithdrawnEvent event) {
        this.balance = event.getBalance();
    }
    ...
```
- He has set all logging to INFO so this is optional (logback.xml):
``` XML
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/base.xml"/>
    <logger name="org.springframework.web" level="INFO"/>
</configuration>
```     
- For the event Handler in plain java the config needs to look like this, also I used slf4j instead of print lines:
``` Java
...
    Configuration config = DefaultConfigurer.defaultConfiguration()
        .configureAggregate(Account.class)
        .configureAggregate(MoneyTransfer.class)
        .registerModule(SagaConfiguration.subscribingSagaManager(MoneyTransferSaga.class))
        .registerModule(new EventHandlingConfiguration().registerEventHandler(configuration -> new LoggingEventHandler()))
        .configureEmbeddedEventStore(c -> new InMemoryEventStorageEngine())
        .buildConfiguration();
...
```
- I removed the changes that give the Saga a UUID as this breaks testability.

###Video 3

- I didn't have remove a lot of my configuration because I am already using the release...
- I recommend that you set logging to `DEBUG` in the logback.xml file and then turn on `spring.jpa.show-sql = true` in the application.yml configuration. This will help you understand what is happening in the code better.
- Remember I selected h2 database? Use the web console to view the Account Balances using `/console` and make sure you have the same Url as the config.
- I have removed the plain java `Application` class as I want to focus on the Spring boot implementation.

___
### Testing

Use http://localhost:8080/console to access the H2 console. (note change port if you using another port)
 - JDBC Url: dbc:h2:mem:axon3db;MODE=Oracle;DB_CLOSE_ON_EXIT=FALSE
 - User name: sa
 - Password: 
 
Then you can use http://localhost:8080/TransactionHistory/1 to see the transaction history for account 1.
and http://localhost:8080/TransferMoney/simple?sourceId=1&destinationId=2&amount=200 to transfer money from account 1 to 2.

 