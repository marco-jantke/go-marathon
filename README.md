[![Build Status](https://travis-ci.org/gambol99/go-marathon.svg?branch=master)](https://travis-ci.org/gambol99/go-marathon)
[![GoDoc](http://godoc.org/github.com/gambol99/go-marathon?status.png)](http://godoc.org/github.com/gambol99/go-marathon)

# Go-Marathon

Go-marathon is a API library for working with [Marathon](https://mesosphere.github.io/marathon/).
It currently supports

- Application and group deployment
- Helper filters for pulling the status, configuration and tasks
- Multiple Endpoint support for HA deployments
- Marathon Event Subscriptions and Event Streams

Note: the library is still under active development; users should expect frequent (possibly breaking) API changes for the time being.

It requires Go version 1.4 or higher.

## Code Examples

There is also an examples directory in the source which shows hints and snippets of code of how to use it —
which is probably the best place to start.

### Creating a client

```Go
import (
	marathon "github.com/gambol99/go-marathon"
)

marathonURL := "http://10.241.1.71:8080"
config := marathon.NewDefaultConfig()
config.URL = marathonURL
client, err := marathon.NewClient(config)
if err != nil {
	log.Fatalf("Failed to create a client for marathon, error: %s", err)
}

applications, err := client.Applications()
...
```

Note, you can also specify multiple endpoint for Marathon (i.e. you have setup Marathon in HA mode and
having multiple running)

```Go
marathonURL := "http://10.241.1.71:8080,10.241.1.72:8080,10.241.1.73:8080"
```

The first one specified will be used, if that goes offline the member is marked as *"unavailable"* and a
background process will continue to ping the member until it's back online.

### Listing the applications

```Go
applications, err := client.Applications()
if err != nil {
	log.Fatalf("Failed to list applications")
}

log.Printf("Found %d applications running", len(applications.Apps))
for _, application := range applications.Apps {
	log.Printf("Application: %s", application)
	details, err := client.Application(application.ID)
	assert(err)
	if details.Tasks != nil && len(details.Tasks) > 0 {
		for _, task := range details.Tasks {
			log.Printf("task: %s", task)
		}
		// check the health of the application
		health, err := client.ApplicationOK(details.ID)
		log.Printf("Application: %s, healthy: %t", details.ID, health)
	}
}
```

### Creating a new application

```Go
log.Printf("Deploying a new application")
application := marathon.NewDockerApplication()
application.Name("/product/name/frontend")
application.CPU(0.1).Memory(64).Storage(0.0).Count(2)
application.Arg("/usr/sbin/apache2ctl", "-D", "FOREGROUND")
application.AddEnv("NAME", "frontend_http")
application.AddEnv("SERVICE_80_NAME", "test_http")
application.AddLabel("environment", "staging")
application.AddLabel("security", "none")
// add the docker container
application.Container.Docker.Container("quay.io/gambol99/apache-php:latest").Expose(80, 443)
application.CheckHTTP("/health", 10, 5)

if _, err := client.CreateApplication(application); err != nil {
	log.Fatalf("Failed to create application: %s, error: %s", application, err)
} else {
	log.Printf("Created the application: %s", application)
}
```

Note: Applications may also be defined by means of initializing a `marathon.Application` struct instance directly. However, go-marathon's DSL as shown above provides a more concise way to achieve the same.

### Scaling application

Change the number of application instances to 4

```Go
log.Printf("Scale to 4 instances")
if err := client.ScaleApplicationInstances(application.ID, 10); err != nil {
	log.Fatalf("Failed to delete the application: %s, error: %s", application, err)
} else {
	client.WaitOnApplication(application.ID, 30 * time.Second)
	log.Printf("Successfully scaled the application")
}
```

### Subscription & Events

Request to listen to events related to applications — namely status updates, health checks
changes and failures. There are two different event transports controlled by `EventsTransport`
setting with the following possible values: `EventsTransportSSE` and `EventsTransportCallback` (default value).
See [Event Stream](https://mesosphere.github.io/marathon/docs/rest-api.html#event-stream) and
[Event Subscriptions](https://mesosphere.github.io/marathon/docs/rest-api.html#event-subscriptions) for details.

#### Event Stream

Only available in Marathon >= 0.9.0. Does not require any special configuration or prerequisites.

```Go
// Configure client
config := marathon.NewDefaultConfig()
config.URL = marathonURL
config.EventsTransport = marathon.EventsTransportSSE

client, err := marathon.NewClient(config)
if err != nil {
	log.Fatalf("Failed to create a client for marathon, error: %s", err)
}

// Register for events
events := make(marathon.EventsChannel, 5)
err = client.AddEventsListener(events, marathon.EVENTS_APPLICATIONS)
if err != nil {
	log.Fatalf("Failed to register for events, %s", err)
}

timer := time.After(60 * time.Second)
done := false

// Receive events from channel for 60 seconds
for {
	if done {
		break
	}
	select {
	case <-timer:
		log.Printf("Exiting the loop")
		done = true
	case event := <-events:
		log.Printf("Recieved event: %s", event)
	}
}

// Unsubscribe from Marathon events
client.RemoveEventsListener(events)
```

#### Event Subscriptions

Requires to start a built-in web server accessible by Marathon to connect and push events to. Consider the following
additional settings:

- `EventsInterface` — the interface we should be listening on for events. Default `"eth0"`.
- `EventsPort` — built-in web server port. Default `10001`.
- `CallbackURL` — custom callback URL. Default `""`.

```Go
// Configure client
config := marathon.NewDefaultConfig()
config.URL = marathonURL
config.EventsInterface = marathonInterface
config.EventsPort = marathonPort

client, err := marathon.NewClient(config)
if err != nil {
	log.Fatalf("Failed to create a client for marathon, error: %s", err)
}

// Register for events
events := make(marathon.EventsChannel, 5)
err = client.AddEventsListener(events, marathon.EVENTS_APPLICATIONS)
if err != nil {
	log.Fatalf("Failed to register for events, %s", err)
}

timer := time.After(60 * time.Second)
done := false

// Receive events from channel for 60 seconds
for {
	if done {
		break
	}
	select {
	case <-timer:
		log.Printf("Exiting the loop")
		done = true
	case event := <-events:
		log.Printf("Recieved event: %s", event)
	}
}

// Unsubscribe from Marathon events
client.RemoveEventsListener(events)
```

A full list of the events:

```Go
const (
	EVENT_API_REQUEST = 1 << iota
	EVENT_STATUS_UPDATE
	EVENT_FRAMEWORK_MESSAGE
	EVENT_SUBSCRIPTION
	EVENT_UNSUBSCRIBED
	EVENT_STREAM_ATTACHED
	EVENT_STREAM_DETACHED
	EVENT_ADD_HEALTH_CHECK
	EVENT_REMOVE_HEALTH_CHECK
	EVENT_FAILED_HEALTH_CHECK
	EVENT_CHANGED_HEALTH_CHECK
	EVENT_GROUP_CHANGE_SUCCESS
	EVENT_GROUP_CHANGE_FAILED
	EVENT_DEPLOYMENT_SUCCESS
	EVENT_DEPLOYMENT_FAILED
	EVENT_DEPLOYMENT_INFO
	EVENT_DEPLOYMENT_STEP_SUCCESS
	EVENT_DEPLOYMENT_STEP_FAILED
	EVENT_APP_TERMINATED
)

const (
	EVENTS_APPLICATIONS  = EVENT_STATUS_UPDATE | EVENT_CHANGED_HEALTH_CHECK | EVENT_FAILED_HEALTH_CHECK | EVENT_APP_TERMINATED
	EVENTS_SUBSCRIPTIONS = EVENT_SUBSCRIPTION | EVENT_UNSUBSCRIBED | EVENT_STREAM_ATTACHED | EVENT_STREAM_DETACHED
)
```

## Contributing

 - Fork it
 - Create your feature branch (git checkout -b my-new-feature)
 - Commit your changes (git commit -am 'Add some feature')
 - Push to the branch (git push origin my-new-feature)
 - Create new Pull Request
 - If applicable, update the README.md
