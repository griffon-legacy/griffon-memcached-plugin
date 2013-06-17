
Memcached support
-----------------

Plugin page: [http://artifacts.griffon-framework.org/plugin/memcached](http://artifacts.griffon-framework.org/plugin/memcached)


The Memcached plugin enables lightweight access to [Memcached][1] databases using [Spymemcached][2] as client.
This plugin does NOT provide domain classes nor dynamic finders like GORM does.

Usage
-----
Upon installation the plugin will generate the following artifacts in `$appdir/griffon-app/conf`:

 * MemcachedConfig.groovy - contains the database definitions.
 * BootstrapMemcached.groovy - defines init/destroy hooks for data to be manipulated during app startup/shutdown.

A new dynamic method named `withMemcached` will be injected into all controllers,
giving you access to a `net.spy.memcached.MemcachedClient` object, with which you'll be able
to make calls to the database. Remember to make all database calls off the EDT
otherwise your application may appear unresponsive when doing long computations
inside the EDT.

This method is aware of multiple databases. If no databaseName is specified when calling
it then the default database will be selected. Here are two example usages, the first
queries against the default database while the second queries a database whose name has
been configured as 'internal'

    package sample
    class SampleController {
        def queryAllDatabases = {
            withMemcached { databaseName, client -> ... }
            withMemcached('internal') { databaseName, client -> ... }
        }
    }

This method is also accessible to any component through the singleton `griffon.plugins.memcached.MemcachedConnector`.
You can inject these methods to non-artifacts via metaclasses. Simply grab hold of a particular metaclass and call
`MemcachedEnhancer.enhance(metaClassInstance, memcachedProviderInstance)`.

Configuration
-------------
### Dynamic method injection

The `withMemcached()` dynamic method will be added to controllers by default. You can
change this setting by adding a configuration flag in `griffon-app/conf/Config.groovy`

    griffon.memcached.injectInto = ['controller', 'service']

### Events

The following events will be triggered by this addon

 * MemcachedConnectStart[config, databaseName] - triggered before connecting to the database
 * MemcachedConnectEnd[databaseName, client] - triggered after connecting to the database
 * MemcachedDisconnectStart[config, databaseName, client] - triggered before disconnecting from the database
 * MemcachedDisconnectEnd[config, databaseName] - triggered after disconnecting from the database

### Multiple Stores

The config file `MemcachedConfig.groovy` defines a default client block. As the name
implies this is the client used by default, however you can configure named clients
by adding a new config block. For example connecting to a server whose name is 'internal'
can be done in this way

    clints {
        internal {
            connectionFactory {
                protocol = CFB.Protocol.TEXT
            }
        }
    }

This block can be used inside the `environments()` block in the same way as the
default client block is used.

### Example

A trivial sample application can be found at [https://github.com/aalmiray/griffon_sample_apps/tree/master/persistence/memcached][3]

Testing
-------
The `withMemcached()` dynamic method will not be automatically injected during unit testing, because addons are simply not initialized
for this kind of tests. However you can use `MemcachedEnhancer.enhance(metaClassInstance, memcachedProviderInstance)` where 
`memcachedProviderInstance` is of type `griffon.plugins.memcached.MemcachedProvider`. The contract for this interface looks like this

    public interface MemcachedProvider {
        Object withMemcached(Closure closure);
        Object withMemcached(String clientName, Closure closure);
        <T> T withMemcached(CallableWithArgs<T> callable);
        <T> T withMemcached(String clientName, CallableWithArgs<T> callable);
    }

It's up to you define how these methods need to be implemented for your tests. For example, here's an implementation that never
fails regardless of the arguments it receives

    class MyMemcachedProvider implements MemcachedProvider {
        Object withMemcached(String clientName = 'default', Closure closure) { null }
        public <T> T withMemcached(String clientName = 'default', CallableWithArgs<T> callable) { null }
    }

This implementation may be used in the following way

    class MyServiceTests extends GriffonUnitTestCase {
        void testSmokeAndMirrors() {
            MyService service = new MyService()
            MemcachedEnhancer.enhance(service.metaClass, new MyMemcachedProvider())
            // exercise service methods
        }
    }


[1]: http://www.memcached.org
[2]: http://code.google.com/p/spymemcached/
[3]: https://github.com/aalmiray/griffon_sample_apps/tree/master/persistence/memcached

