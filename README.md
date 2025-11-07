# powerwall3mqtt
A simple service that acts as a bridge between the Powerwall 3 TEDAPI and MQTT.
Scheme is compatible with Home Assistant's MQTT integration.

## Fork for standalone deployment
The upstream version is meant to be used as a Home Assistant Add On, which
limits it to use primarily with Home Assistant OS (HAOS). This fork allows it to
be deployed generally. This fork is almost entirely composed additional files;
only tiny modifications are made to the upstream code. Specifically, this fork
builds `powerwall3mqtt` with Bazel, which has reproducible builds, and is able
to build a multi-architecture OCI image (amd64 and arm64). Note that Bazel
does _not_ use the `Dockerfile`.

## Running the image
You will need to set some environment variables so that `powerwall3mqtt` knows
how to authenticate to your Powerwall 3 and how to talk to your MQTT broker.

Here are the most notable environment variables:
```sh
POWERWALL3MQTT_CONFIG_TEDAPI_PASSWORD="Password_printed_on_inside_of_Powerwall"
POWERWALL3MQTT_CONFIG_MQTT_HOST="hostname.or.ip.address.of.mqtt.broker"
```

There are more options if you need them. See `config.yaml` for all the settings,
and tweak their names to follow the pattern above.

See the `deploy` directory for example configs.

## Development
The build uses Bazel. This is simple to do and does not require installing
anything to your computer other than Bazelisk, the Bazel wrapper. Bazelisk is a
single static binary which will download and execute the correct version of
Bazel for you. It installs Bazel in `~/.cache/bazelisk` so does not require root
privilege. Simply download the binary from
[here](https://github.com/bazelbuild/bazelisk/releases), name it `bazel`, and
put it somewhere in your `PATH` (e.g. `/usr/local/bin`).

### Running the program locally
For development, you can easily run the binary locally without using a container
image (but it will still use the correct version of all dependencies):

```sh
bazel run //app:powerwall3mqtt
```

### Building the image
You can build the multi-arch image like so:

```sh
bazel build //app:powerwall3mqtt-image-index
```

It will place the image in `bazel-bin/app/powerwall3mqtt-image-index`; from
there you can use your local tooling (e.g. Docker or Podman) to load and run the
image.

If you just want to push the image to a container image repository, you can do
something simpler. Edit `app/BUILD.bazel`, find the line which says
`repository`, and edit it for your repo. Then do this:

```sh
bazel run //app:powerwall3mqtt-push
```

## Current state
- Powerwall 3 local access requires a direct connection to the TeslaPW_XXXXX WiFi network.  The device running this add-on must have an IP from that network, which is 192.168.91.x/24.  A wired ethernet connection **will not work**.
- The bridge can deal with a single group of one or more Powerwall 3s.  It might also support expansion units, but I have none to test with.
- Power reporting is working for the following:
	- Aggregates of the entire system
	- Individual PV strings on each PW3
- Energy storage reporting is working for the following:
	- Aggregate of all batteries
	- Individual Powerwall battery levels
	- Calculations of percentage remaining and user defined backup reserve mirror the Tesla app
- Polling interval defaults to 30 seconds
	- It can be configured down to 1 second, but Powerwalls may tell the add-on to backoff.
	- Issues related to sporadic data will only be looked at if the interval is set to at least 5.
- No other energy reporting
	- Pypowerwall doesn't have it yet, and so far it appears it may not be possible
	- The Energy Dashboard can be supported by manually creating 6 Integral helpers.  The steps as of 2025-02-24 are:
		- Enable the 4 power input/output entities listed below.  They are disabled by default.
			- Battery Power Charge
			- Battery Power Discharge
			- Grid Power Import
			- Grid Power Export
		- Create 6 Integral Helpers in Settings, Integrations, Helpers
			- Each should be a Left Riemann sum
			- You should create one for each for the following sensors:
				- Battery Power Charge (name it Battery Energy Charged)
				- Battery Power Discharge (name it Battery Energy Discharged)
				- Grid Power Import (name it Grid Energy Imported)
				- Grid Power Export (name it Grid Energy Exported)
				- Load Power (name it Load Energy Used)
				- Solar Power (name is Solar Energy Produced)
- No interactivity support
	- Can't tell the system to go Off-Grid
	- Can't change settings on the Powerwall
	- I'm planning to work on these later this year when I'm on-site where my PW3 install is

## Pre-reqs
**Starting with firmware 25.10.1, the only known way to communicate with the TEDAPI is using the TeslaPW_XXXXX WiFi network.**
If your HA server is within range of the PW you can join it directly, but otherwise you'd need to create some kind of bridge, which is beyond the scope of this project.

## Reporting problems
- Please switch the Logging Level to DEBUG in Configuration and restart the add-on.
	- You may need to toggle "Show unused optional configuration options" on.
- Open an issue at https://github.com/slyglif/powerwall3mqtt/issues
	- Describe the issue you are encountering
	- Include the version of HA and HAOS
	- Include the version of the add-on
	- Include the IP address for HAOS.  Problem reports without this will be closed.
	- Include the logs
		- Include the full logs if possible
		- If the full logs are megabytes in size, at least the first 20 lines at startup and the 10 lines prior to the error
		- If including the logs inline, please quote them between "\`\`\`" marks (3 backticks)
