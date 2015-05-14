Transactions are where Objectify departs most significantly from the patterns set by the Low-Level API and JDO/JPA.  Objectify's transactions are inspired by [EJB](http://en.wikipedia.org/wiki/Enterprise_JavaBeans) and [NDB](https://developers.google.com/appengine/docs/python/ndb/transactions) transactions: Designed to allow modular, convenient transactional logic with a minimum of boilerplate.

You should familiarize yourself with the [GAE Transactions](https://developers.google.com/appengine/docs/java/datastore/transactions) documentation.

# Basic Transactions #

A basic transaction looks like this:

```
import static com.googlecode.objectify.ObjectifyService.ofy;
import com.googlecode.objectify.Work;

// If you don't need to return a value, you can use VoidWork
Thing th = ofy().transact(new Work<Thing>() {
    public Thing run() {
        Thing thing = ofy().load().key(thingKey).now();
        thing.modify();
        ofy().save().entity(thing);
        return thing;
    }
});
```

Note that all pending async operations are automatically completed at the end of a transaction.

## Idempotence ##

**Work MUST be idempotent.**  A variety of conditions, including `ConcurrentModificationException`, can cause a transaction to retry.  If you need to limit the number of times a transaction can be tried, use `transactNew(int, Work)`.

## Cross-Group Transactions ##

Objectify requires no special flags to enable cross-group transactions.  If you access more than one entity group in a transaction, the transaction with be an XG transaction.  If you do access only one, it is not.  The standard limit of 5 EGs applies to all transactions.

## Transaction Inheritance ##

Transaction contexts are inherited.  In this example, the inner `transact()` does **not** start a new transaction:

```
public void doSomeWork() {
    ofy().transact(new VoidWork() {
        public void vrun() {
            doSomeMoreWork();
        }
    }
}

public void doSomeMoreWork() {
    ofy().transact(new VoidWork() {
        public void vrun() {
            // When called from doSomeWork(), executes in the original transaction context
            // When called from outside a transaction, executes in a new transaction
        }
    }
}
```

This makes it easy to create modular bits of transactional logic that can be used from a variety of contexts without having to pass around `Objectify` instances as parameters.

If you need to suspend a transaction and begin a new one, use the `transactNew()` method:

```
ofy().transactNew(new VoidWork() {
    public void vrun() {
        Thing thing = ofy().load().key(thingKey).now();
        thing.modify();
        ofy().save().entity(thing);
    }
});
```

The old transaction (if present) is suspended for the duration of the new transaction.  After the transaction commits (or rolls back), the original transaction will resume.

## Escaping Transactions ##

There are often times in the middle of a transaction when you would like to make a datastore request outside of a transaction.  Perhaps you wish to make a query without an ancestor(), or perhaps you wish to load an entity without enlisting it in a XG transaction.

Objectify makes it easy to run operations outside of a transaction:

```
ofy().transact(new VoidWork() {
    public void vrun() {
        Thing thing = ofy().load().key(thingKey).get();
        Other other = ofy().transactionless().load().key(thing.otherKey).now();
        if (other.specialState) {
            thing.modify();
            ofy().save().entity(thing);
        }
    }
});
```

This circumstance doesn't come up often but it does come up.

## Transactions and Caching ##

Starting a transaction creates a new `Objectify` instance with a fresh, empty session cache.  Loads and saves will populate this new instance cache; because of this, entities modified will appear modified after being loaded again within the same transaction.  Unlike the low-level API, the datastore will not appear "frozen in time", although transaction isolation is maintained.

When a transaction successfully commits, its session cache will be copied into the parent `Objectify` instance's session cache.

Transactions integrate correctly with Objectify's global memcache.  Reads and writes bypass the memcache, but a successful commit will reset the memcache value for changed entities.

# execute() #

Objectify provides a method designed to facilitate EJB-like behavior:

```
ofy().execute(TxnType.REQUIRED, new VoidWork() {
    public void vrun() {
        Thing thing = ofy().load().key(thingKey).now();
        thing.modify();
        ofy().save().entity(thing);
    }
});
```

The `TxnType` enum provides the common [EJB transaction attributes](http://en.wikipedia.org/wiki/Enterprise_JavaBeans#Transactions):

| MANDATORY | If not already in a transaction, throw an exception.  Otherwise, use the transaction. |
|:----------|:--------------------------------------------------------------------------------------|
| REQUIRED | If there is already a transaction in progress, use it.  If not, start a new transaction. |
| REQUIRES\_NEW | If there is already a transaction in progress, suspend it.  Always start a new transaction. |
| SUPPORTS | If there is a transaction in progress, use it.  Otherwise, execute without a transaction. |
| NOT\_SUPPORTED | If there is a transaction in progress, suspend it.  Execute without a transaction. |
| NEVER | If there is a transaction in progress, throw an exception.  Otherwise, execute without a transaction. |

You will probably not use this method directly.  However, you can easily use it to build AOP interceptors for Guice and Spring.

# Transactions with Guice #

Using Guice AOP you can place EJB-like annotations on methods and eliminate all the `new Work<Blah>() {...`} boilerplate:

```
public class Worker {
    @Transact(TxnType.REQUIRED)
    public void doSomeWork() {
        ofy().load()...    // do some work
        Thing th = doSomeMoreWork();
        ofy().save()...    // do some work
    }

    @Transaction(TxnType.REQUIRED)
    public Thing doSomeMoreWork() {
        ofy().load()...   // do some more work
        return someThing;
    }
}
```
```
Worker worker = injector.getInstance(Worker.class);
worker.doSomeWork();   // all work done in a single transaction!
```

These methods are automatically wrapped in `Work` classes and executed with `Objectify.execute()`.

This requires two classes:

```
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface Transact {
    TxnType value();
}
```
```
public class TransactInterceptor implements MethodInterceptor {

    /** Work around java's annoying checked exceptions */
    private static class ExceptionWrapper extends RuntimeException {
        private static final long serialVersionUID = 1L;

            public ExceptionWrapper(Throwable cause) {
                super(cause);
            }
		
            /** This makes the cost of using the ExceptionWrapper negligible */
            @Override
            public synchronized Throwable fillInStackTrace() {
                return this;
            }
        }

    /** The only trick here is that we need to wrap & unwrap checked exceptions that go through the Work interface */
    @Override
    public Object invoke(final MethodInvocation inv) throws Throwable {
        Transact attr = inv.getStaticPart().getAnnotation(Transact.class);
        TxnType type = attr.value();

        try {
            return ofy().execute(type, new Work<Object>() {
                @Override
                public Object run() {
                    try {
                        return inv.proceed();
                    }
                    catch (RuntimeException ex) { throw ex; }
                    catch (Throwable th) { throw new ExceptionWrapper(th); }
                }
            });
        } catch (ExceptionWrapper ex) { throw ex.getCause(); }
    }
}
```

Now, to enable this interceptor, add this to your Guice module configuration:

```
public static class YourModule extends AbstractModule {
    @Override
    protected void configure() {
        bindInterceptor(Matchers.any(), Matchers.annotatedWith(Transact.class), new TransactInterceptor());
        // continue with your guice configuration
    }
}
```

You can find a working example of this in the [Motomapia](http://www.motomapia.com/) sample application.

# Transactions with Spring #

With Spring AOP transaction management can be implemented as a cross-cutting _aspect_ to be applied to  [point-cuts](http://docs.spring.io/spring/docs/2.0.x/reference/aop.html) in the application.

Imagine you have the following test worker, for which you want all methods to run in a transaction:

```
@Component
public class TestWorker {
    public void doSomething() {
    	ofy().save().entity(new TestEntity(id)).now();
   }
    public TestEntity doSomethingElse() {
    	return ofy().load().key(testKey).now();
   }
   ...
}
```

When injecting the bean:

```
@Resource
private TestWorker testWorker;
```

For every method call, either a separate transaction should be opened or the preexisting transactional context should be inherited:

```
testWorker.doSomething();

testWorker.doSomethingElse();
```

To achieve this, two things need to be done: firstly, one has to implement an _around advice_ in order to wrap every method invocation in a `Work` instance and execute it with `ofy().transact()`:

```
public class AppEngineTransactionManager {

    public Object transact(final ProceedingJoinPoint joinpoint) throws Throwable {
        try {
            return ofy().transact(new Work<Object>() {
		             
                public Object run() {
                    try {
                        /** The methods will be executed in the transact() method*/
                        return joinpoint.proceed();
                    } catch (Throwable t) {
                        /** Don't forget to wrap checked exceptions that go through the Work interface */
                        throw new ExceptionWrapper(t);
                    }
                }
            });
        } catch (ExceptionWrapper e) {
            /** Don't forget to unwrap checked exceptions that went through the Work interface */
            throw e.getCause();
        } catch (Throwable t) {
            throw new RuntimeException("Unexpected exception during transaction management", t);
        }
    }

    private static class ExceptionWrapper extends RuntimeException {
        ...
    }
}
```

Secondly, one has to declare the _aspect_ with the _around advice_ in the Spring configuration:

```
<!-- The bean that will be used for AOP advice -->
<bean id="appEngineTransactionManager" class="com.example.AppEngineTransactionManager"/>

<aop:config>
    <aop:aspect ref="appEngineTransactionManager">
        <!-- Match all methods of all classes ending with Worker -->
        <aop:pointcut id="workMethod" expression="execution(* *..*Worker.*(..))"/>
        <!-- Invoke the around advice whenever the pointcut matches -->
        <aop:around pointcut-ref="workMethod" method="transact"/>
    </aop:aspect>
</aop:config>
```

A working prototype can be found here:
https://github.com/dajudge/appengine-objectify-spring-transactions-example