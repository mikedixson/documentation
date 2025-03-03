.. _Bootstrapping:

=============
Bootstrapping
=============

Every php process has a relatively short lifespan that lasts as long as the HTTP request or the invokation of the command
line program. At the beginning of this lifespan, Nextcloud initializes its services. At the same time, any additional apps
might want to register their services to Nextcloud as well. This event is called the *bootstrapping* and this chapter
shall shed some light on how to hook into this with an app.


.. _application-php:

The Application class
---------------------

The `Application` class is the main entry point of an app. This class is optional but highly recommended if your app needs
to register any services or run some code for every request.


Nextcloud will try to autoload the class from the namespace ``\OCA\<App namespace>\AppInfo\Application``, like e.g.
``\OCA\Mail\AppInfo\Application``. The file will therefore be located at ``lib/AppInfo``.

.. code-block:: php

    <?php

    declare(strict_types=1);

    namespace OCA\MyApp\AppInfo;

    use OCP\AppFramework\App;

    class Application extends App {

        public function __construct() {
            parent::__construct('myapp');
            \OCP\Util::connectHook('OC_User', 'pre_deleteUser', 'OCA\MyApp\Hooks\User', 'deleteUser');
        }

    }

The class **must** extend ``OCP\AppFramework\App`` and it may optionally implement ``\OCP\AppFramework\Bootstrap\IBootstrap``:

.. code-block:: php

    <?php

    declare(strict_types=1);

    namespace OCA\MyApp\AppInfo;

    use OCA\MyApp\Listeners\UserDeletedListener;
    use OCA\MyApp\Notifications\Notifier;
    use OCP\AppFramework\App;
    use OCP\AppFramework\Bootstrap\IBootstrap;
    use OCP\Notification\IManager;
    use OCP\User\Events;

    class Application extends App implements IBootstrap {

        public function register(IRegistrationContext $context): void {
            // ... registration logic goes here ...

            $context->registerEventListener(
                BeforeUserDeletedEvent::class,
                UserDeletedListener::class
            );
        }

        public function boot(IBootContext $context): void {
            // ... boot logic goes here ...

            /** @var IManager $manager */
            $manager = $context->getAppContainer()->query(IManager::class)
            $manager->registerNotifierService(Notifier::class);
        }

    }

Bootstrapping process
---------------------

To give a better overview of *when* each of the bootstrapping stages are reached and how they app can interact with them,
this section explains the changes done for Nextcloud 20.

Nextcloud 20 and later
**********************

Nextcloud 20 is the first release with the interface ``\OCP\AppFramework\Bootstrap\IBootstrap``. This interface can be
implemented by apps' Application class to signal that they want to act on the bootstrapping stages. The major difference
between this and the old process is that the boostrapping is not performed in a sequence, but apps register and boot
interleaved. This should ensure that an app that boots can rely on all other apps' registration to be finished.

The overall process is as follows.

1) Each installed and enabled app that has an ``Application`` class class that implements ``IBootstrap``, the ``register``
   method will be called. This method receives a context argument via which the app can prime the dependency injection
   container and register many other services lazily. The emphasis is on **lazyness**. At this very early stage of the
   process lifetime, no other apps nor all of the server components are ready. Therefore the app **must not** try to use
   anything except the API provided by the context. That shall ensure that all apps can safely run their registration logic
   before any services are queried from the DI container or related code is run.
2) Nextcloud will load groups of certain apps early, like filesystem or session apps, and other later. For this their optional
   :ref:`app-php` will be included. As ``app.php`` is deprecated, apps should try not to rely on this step.
3) Nextcloud will query the app's ``Application`` class (again), no matter if it implements ``IBootstrap`` or not.
4) Nextcloud will invoke the ``boot`` method of every ``Application`` instance that implements ``IBootstrap``. At this stage
   you may assume that all registrations via ``IBootstrap::register`` have been done.

Nextcloud 19 and older
**********************

Nextcloud will load groups of certain apps early, like filesystem or session apps, and other later. For this their optional
:ref:`app-php` will be included. The ``Application`` class is only queried for some requests, so there is no guarantee that
its contstructor will be invoked.


.. _app-php:

app.php (deprecated)
--------------------

Nextcloud will ``require_once`` every installed and enabled app's ``appinfo/app.php`` file if it exists. The app can use
this file to run registrations of services, event listeners and similar.

To leverage the advantages of object-oriented programming, it's recommended to put the logic into an :ref:`Application<application-php>`
class and query an instance like

.. code-block:: php

    <?php

    declare(strict_types=1);

    $app = \OC::$server->query(\OCA\MyApp\AppInfo\Application::class);
