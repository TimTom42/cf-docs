---
title: Health Manager
---

The Health Manager monitors the state of applications deployed to Cloud Foundry and ensures that started applications are running, and that their versions and number of instances are correct.

The Health Manger compares the actual state of an application to its desired state, and when it detects a discrepancy, it initiates actions to return the application to the desired state. For instance, if fewer than expected instances are running, the Health Manager will instruct the appropriate DEA to start the appropriate number of instances.    

The actual state of an application is represented by an instance of the `AppState` class, which updates itself based on heartbeats and `droplet.exited` messages issued by the DEA running the application. The desired state of an application is obtained from a dump of the Cloud Controller database.


## <a id='components'></a>Health Manager Components ##

This section describes Health Manager components.

* Manager --- The Manager component provides an entry point and configures, initializes and registers other Health Manager components.
* Harmonizer --- The Harmonizer periodically compares the actual state of an application to the desired state; the Scheduler and Nudger actions are used to bring the states into harmony.
* Scheduler --- The Scheduler encapsulates Ruby EventMachine-related functionality such as timer setup and cancellation, and quantization of long-running tasks to prevent Event Machine Reactor loop blocking.
* Desired State --- The Desired State component obtains the expected state of the application: stopped or started, how many instances should be running, and so on, from the [Cloud Controller](./cloud-controller.html) via the HTTP-based Bulk API. The Bulk API returns a dump of the CC_DB database, which contains the Cloud Controller's expected state for all applications. 
* Actual State --- The Actual State component listens to heartbeat and other messages on the [NATS](./messaging-nats.html) bus from DEAs. 
* Nudger --- Nudger is the interface that the Health Manager uses to remedy discrepencies between desired and actual application state by dispatching `cloudcontrollers.hm.requests` messages that instruct Cloud Controller nodes to start or stop instances. Nudger maintains a priority queue of these requests, and dequeues the messages by a batchful.
* Reporter --- The Reporter responds to `healthmanager.status` and `healthmanager.health` requests.


## <a id='harmonization'></a>Harmonization Policy ###

Conceptually, harmonization happens in two ways:

- By reacting to messages (such as `droplet.exited`)
- By periodically scanning the world and enumerating applications, looking for anomalies.

### <a id='droplet-exited'></a>Handling droplet exited Messages ###

There are three scenarios that result in a `droplet.exited` message:

- An  application was stopped explicitly, in which case no remedial action is required. 
- A DEA is being evacuated and all application instances running there need to be restarted somewhere else. Health Manager initiates the restarts.
- An application instance crashed. The crashed instance needs to be restarted, unless it crashed multiple times within a short period of time, in which case it is declared `flapping`. See the following section for more information.

### <a id='flapping-instances'></a>Handling Flapping Instances ###

An instance of application is declared "flapping" if it crashed more than `flapping_death` times within `flapping_timeout` seconds. There are several possible reasons for flapping:

- The application is completely broken and will not start.
- The application has a bug that results in repeated crashes.
- The application depends on an external or Cloud Foundry-provisioned service that is unavailable, resulting in repeated crashes.

The goals for handling a flapping application are: 

- Attempt to restart an application when appropriate.
- Provide crashlogs for crashed instances.
- Reduce the overhead associated with restarting an application, particularly the overhead associated with moving application bits to the DEA.
- Avoid IO spikes due to massive simultaneous restarts.

To accomodate these conflicting requirements, the policy for handling a flapping instance is:

- Initially restart the instance with a delay defined by `min_restart_delay`.
- For each subsequent crash, double the delay up to the value of`max_restart_delay`.
- Add a random noise to the value of delay, its maximum absolute value defined by `delay_time_noise`.
- If the number of crashes for flapping instance exceeds `giveup_crash_number`, stop trying to restart the instance. This behavior can be disabled.

### <a id='heartbeat-processing'></a>Heartbeat Processing ###

DEAs periodically issue heartbeat messages on the NATS bus. A heartbeat message identifies the issuing DEA and includes information about application instances running on the DEA.

The heartbeats are used to establish whether there are "missing" or "extra" instances. An `AppState` object for each instance of each version.

An instance is "missing" if its `AppState` object has not received a heartbeat in the last `droplet_lost` seconds. However, an `instance_missing` event is triggered only if `AppState` was not reset recently, and if `check_for_missing_instances` method has been invoked.

An `instance_missing` event triggers a start command. A stop command is triggered for each extra instance.


## <a id='configuration'></a>Health Manager Configuration ##

The Health Manager is configured in the `health_manager.yml` file. See [example config file](https://github.com/cloudfoundry/health_manager/blob/master/config/health_manager.yml) for an explanation of all the configurable variables.

## <a id='logging'></a>Heatlh Manager Logging ##

Health Manager uses [Steno](http://github.com/cloudfoundry/steno) to manage its logs. The `logging` key in `health_manager.yml` specifies the log level. The table below lists and defines log levels; the right-hand column provides examples of occurrences associated with a log level.:

| Log Level| Description  | Example Events |
| :-------- | :---------- |:---------- |
|error |Situation that might need corrective action.  |Health Manager received an error response from the Cloud Controller bulk API. |
|warn |Situation that might result in an error.| A droplet analysis was initiated while the previous droplet analysis was still running.|
|info |Records a completed action or the current state of a component. |Health Manager registered a new component; Health Manager is shutting down. |
|debug |Records steps in a complex task. |Health Manager starts or stops an instance.  |
|debug2 |Records steps in a complex task at a granular level of detail. |Health Manager received a heartbeat from a DEA; Health Manager compares an application's desired and known states. |

