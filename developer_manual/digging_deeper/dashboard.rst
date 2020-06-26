=========
Dashboard
=========

The dashboard app aims to provide the user with a general overview of their
Nextcloud and shows information that is currently important.

App developers can integrate into the dashboard app and provide their own panels.


Register a dashboard panels
---------------------------

A dashboard panel is represented by a class implementing the `OCP\\Dashboard\\IPanel`
interface. This class will be instantiated whenever the dashboard is loaded.
Any bootstrap code that is needed for the panel can be implemented inside
of the `load` method and will be called when the dashboard is loaded.

.. code-block:: php

    <?php

    namespace OCA\MyApp\Dashboard;

    use OCP\Dashboard\IPanel;
    use OCP\IInitialStateService;
    use OCP\IL10N;
    use OCP\IURLGenerator;
    use OCP\IUserSession;

    class MyAppPanel implements IPanel {

        public function __construct(
            IInitialStateService $initialStateService,
            IL10N $l10n,
            IURLGenerator $urlGenerator
        ) {
            $this->initialStateService = $initialStateService;
            $this->l10n = $l10n;
            $this->urlGenerator = $urlGenerator;
        }

        /**
         * @return string Unique id that identifies the panel, e.g. the app id
         * @since 20.0.0
         */        public function getId(): string {
            return 'myapppanelid';
        }

        /**
         * @return string User facing title of the panel
         * @since 20.0.0
         */
        public function getTitle(): string {
            return $this->l10n->t('My app');
        }

        /**
         * @return int Initial order for panel sorting
         * @since 20.0.0
         */
        public function getOrder(): int {
            return 0;
        }

        /**
         * @return string css class that displays an icon next to the panel title
         * @since 20.0.0
         */
        public function getIconClass(): string {
            return 'icon-class';
        }

        /**
         * @return string|null The absolute url to the apps own view
         * @since 20.0.0
         */
        public function getUrl(): string {
            return $this->urlGenerator->linkToRouteAbsolute('myapp.view.index');
        }

        /**
         * Execute panel bootstrap code like loading scripts and providing initial state
         */
        public function load(): void {
            $initialStateService->provideInitialState('myapp', 'myData', []);
            \OCP\Util::addScript('myapp', 'dashboard');
        }
    }


The class needs to be registered during the app bootstrap.

.. code-block:: php

    <?php

    declare(strict_types=1);

    namespace OCA\MyApp\AppInfo;

    use OCP\AppFramework\App;
    use OCP\AppFramework\Bootstrap\IBootContext;
    use OCP\AppFramework\Bootstrap\IBootstrap;
    use OCP\AppFramework\Bootstrap\IRegistrationContext;
    use OCA\MyApp\Dashboard\MyAppPanel;

    class Application extends App implements IBootstrap {

        public const APP_ID = 'myapp';

        public function __construct(array $urlParams = []) {
            parent::__construct(self::APP_ID, $urlParams);
        }

        public function register(IRegistrationContext $context): void {
            $context->registerDashboardPanel(MyAppPanel::class);
        }

        public function boot(IBootContext $context): void {
        }
    }

For compatibility reasons the panel registration can also be performed by
listening to the `OCP\\Dashboard\\RegisterPanelEvent` for apps that still
need to support older versions where the new app boostrap flow is not available,
however this method is deprecated and will be removed once Nextcloud 19 is EOL.

.. code-block:: php

    use OCP\Dashboard\RegisterPanelEvent;
    use OCP\EventDispatcher\IEventDispatcher;

    class Application extends App {
        public function __construct(array $urlParams = []) {
            parent::__construct(self::APP_ID, $urlParams);
            $container = $this->getContainer();

            /** @var IEventDispatcher $dispatcher */
            $dispatcher = $container->getServer()->query(IEventDispatcher::class);
            $dispatcher->addListener(RegisterPanelEvent::class, function (RegisterPanelEvent $event) use ($container) {
                    \OCP\Util::addScript('myapp', 'dashboard');
                    $event->registerPanel(MyAppPanel::class);
            });
        }
    }


Provide a user interface
------------------------

The user interface can be registered though the public `OCA.Dashboard.register`
JavaScript method. The first parameter represents the panel id that has already
been specified in the `IPanel` implementation. The callback parameter will be
called to render the panel in the frontend. The user interface can be added to
the provided DOM element `el`.

The following example shows how a Vue.js component could be used to render the
panel user interface, however this approach works for any other framework as well
as plain JavaScript as well:


.. code-block:: javascript

    import Dashboard from './components/Dashboard.vue'

    document.addEventListener('DOMContentLoaded', () => {
        OCA.Dashboard.register('myapppanelid', (el) => {
            const View = Vue.extend(Dashboard)
            const vm = new View({
                propsData: {},
                store,
            }).$mount(el)
        })
    })

