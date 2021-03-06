Configuring MassTransit
=======================

.. attention:: **This page is obsolete!**

   New documentation is located at http://masstransit-project.com/MassTransit.

   The latest version of this page can be found here_.

.. _here: http://masstransit-project.com/MassTransit/usage/configuration.html

To configure MassTransit in your application, it depends upon the application type.
There are several application types, which are covered below. In these examples, the
use of a dependency injection framework is not included. Using a container (such as
Autofac, StructureMap, etc.) with MassTransit is covered separately.


Configuring MassTransit in a console application
------------------------------------------------

A console application has a ``Main`` entry point, which is part of the ``Program.cs`` class
by default. The example below configures a simple bus instance that publishes an event with
a value entered.

.. sourcecode:: csharp
    :linenos:

    namespace EventPublisher
    {
        public interface ValueEntered
        {
            string Value { get; }
        }

        public class Program
        {
            public void static Main()
            {
                var busControl = ConfigureBus();
                busControl.Start();

                do
                {
                    Console.WriteLine("Enter message (or quit to exit)");
                    Console.Write("> ");
                    string value = Console.ReadLine();

                    if("quit".Equals(value, StringComparison.OrdinalIgnoreCase))
                        break;

                    busControl.Publish<ValueEntered>(new
                    {
                        Value = value
                    });
                }
                while (true);
                
                busControl.Stop();
            }

            static IBusControl ConfigureBus()
            {
                return Bus.Factory.CreateUsingRabbitMq(cfg =>
                {
                    cfg.Host(new Uri("rabbitmq://localhost"), h =>
                    {
                        h.Username("guest");
                        h.Password("guest");
                    });
                });
            }
        }
    }

In the example, the bus is configured and started, after which a publishing loop
allows values to be entered and published. When the loop exits, the ``busControl``
variable is disposed, which stops the bus.


Configuring MassTransit in a Windows service
--------------------------------------------

A Windows service is recommended for consuming commands and events as it provides an
autonomous execution environment for message consumers. The service can be started and
stopped using the service control manager, as well as monitored by operations tools.

.. note::

    To create a Windows service, we strongly recommend using Topshelf, as it was built
    specifically for this purpose. Topshelf is easy to use, has zero dependencies, and
    creates a service that can be self-installed without additional tools.

The important aspect of configuring a bus in a Windows service is to ensure that the bus
is only configured and started when the service is *started*.

.. sourcecode:: csharp
    :linenos:

    namespace EventService
    {
        using Topshelf;

        public class Program
        {
            public int static Main()
            {
                return HostFactory.Run(cfg => cfg.Service(x => new EventConsumerService());
            }
        }

        class EventConsumerService :
            ServiceControl
        {
            IBusControl _bus;

            public bool Start(HostControl hostControl)
            {
                _bus = ConfigureBus();
                _bus.Start();

                return true;
            }

            public bool Stop(HostControl hostControl)
            {
                _bus?.Stop(TimeSpan.FromSeconds(30));

                return true;
            }

            IBusControl ConfigureBus()
            {
                return Bus.Factory.CreateUsingRabbitMq(cfg =>
                {
                    var host = cfg.Host(new Uri("rabbitmq://localhost"), h =>
                    {
                        h.Username("guest");
                        h.Password("guest");
                    });

                    cfg.ReceiveEndpoint(host, "event_queue", e =>
                    {
                        e.Handler<ValueEntered>(context =>
                            Console.Out.WriteLineAsync($"Value was entered: {context.Message.Value}"));
                    })
                });
            }
        }
    }


Configuring MassTransit in a web application
--------------------------------------------

Configuring a bus in a web site is typically done to publish events, send commands,
as well as engage in request/response conversations. Hosting receive endpoints and
persistent consumers is not recommended (use a service as shown above).

In a web application, the ``HttpApplication`` class methods of Application_Start and
Application_End are used to configure/start the bus and stop the bus respectively.

.. note::

    While many MassTransit samples use Topshelf, web applications are an exception
    where the standard web application conventions are followed.

.. sourcecode:: csharp

    public class MvcApplication :
        HttpApplication
    {
        static IBusControl _busControl;

        public static IBus Bus
        {
            get { return _busControl; }
        }

        protected void Application_Start()
        {
            _busControl = ConfigureBus();
            _busControl.Start();
        }

        protected void Application_End()
        {
            _busControl.Stop(TimeSpan.FromSeconds(10));;
        }

        IBusControl ConfigureBus()
        {
            return Bus.Factory.CreateUsingRabbitMq(cfg =>
            {
                var host = cfg.Host(new Uri("rabbitmq://localhost"), h =>
                {
                    h.Username("guest");
                    h.Password("guest");
                });
            });
        }
    }

    public class NotifyController :
        Controller
    {
        public async Task<ActionResult> Put(string value)
        {
            await MvcApplication.Bus.Publish<ValueNotified>(new
            {
                Value = value
            });

            return View();
        }
    }

    public class CommandController :
        Controller
    {
        public async Task<ActionResult> Send(string value)
        {
            var endpoint = await MvcApplication.Bus.GetSendEndpoint(_serviceAddress);

            await endpoint.Send<SubmitValue>(new
            {
                Timestamp = DateTime.UtcNow,
                Value = value
            });

            return View();
        }
    }

The above example is kept simple, providing a static ``MvcApplication.Bus`` property
to access the bus instance (for publishing events, and sending commands to endpoints).
Newer version of ASP.NET have built-in dependency resolution, in which case the ``IBus``
should be registered so that controllers can specify the dependency in the constructor.
In fact, the inherited ``IPublishEndpoint`` and ``ISendEndpointProvider`` should also
be registered.

The example controllers show how to publish and send messages as well.
