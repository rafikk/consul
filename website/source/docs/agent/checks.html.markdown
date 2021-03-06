---
layout: "docs"
page_title: "Check Definition"
sidebar_current: "docs-agent-checks"
description: |-
  One of the primary roles of the agent is management of system- and application-level health checks. A health check is considered to be application-level if it is associated with a service. A check is defined in a configuration file or added at runtime over the HTTP interface.
---

# Checks

One of the primary roles of the agent is management of system-level and application-level health
checks. A health check is considered to be application-level if it is associated with a
service. If not associated with a service, the check monitors the health of the entire node.

A check is defined in a configuration file or added at runtime over the HTTP interface.  Checks
created via the HTTP interface persist with that node.

There are three different kinds of checks:

* Script + Interval - These checks depend on invoking an external application
  that performs the health check, exits with an appropriate exit code, and potentially
  generates some output. A script is paired with an invocation interval (e.g.
  every 30 seconds). This is similar to the Nagios plugin system.

* HTTP + Interval - These checks make an HTTP `GET` request every Interval (e.g.
  every 30 seconds) to the specified URL. The status of the service depends on the HTTP response code:
  any `2xx` code is considered passing, a `429 Too Many Requests` is a warning, and anything else is a failure.
  This type of check should be preferred over a script that uses `curl` or another external process
  to check a simple HTTP operation. By default, HTTP checks will be configured
  with a request timeout equal to the check interval, with a max of 10 seconds.
  It is possible to configure a custom HTTP check timeout value by specifying
  the `timeout` field in the check definition.

* <a name="TTL"></a>Time to Live (TTL) - These checks retain their last known state for a given TTL.
  The state of the check must be updated periodically over the HTTP interface. If an
  external system fails to update the status within a given TTL, the check is
  set to the failed state. This mechanism, conceptually similar to a dead man's switch,
  relies on the application to directly report its health. For example, a healthy app
  can periodically `PUT` a status update to the HTTP endpoint; if the app fails, the TTL will
  expire and the health check enters a critical state.

## Check Definition

A script check:

```javascript
{
  "check": {
    "id": "mem-util",
    "name": "Memory utilization",
    "script": "/usr/local/bin/check_mem.py",
    "interval": "10s"
  }
}
```

A HTTP check:

```javascript
{
  "check": {
    "id": "api",
    "name": "HTTP API on port 5000",
    "http": "http://localhost:5000/health",
    "interval": "10s",
    "timeout": "1s"
  }
}
```

A TTL check:

```javascript
{
  "check": {
    "id": "web-app",
    "name": "Web App Status",
    "notes": "Web app does a curl internally every 10 seconds",
    "ttl": "30s"
  }
}
```

Each type of definition must include a `name` and may optionally
provide an `id` and `notes` field. The `id` is set to the `name` if not
provided. It is required that all checks have a unique ID per node: if names
might conflict, unique IDs should be provided.

The `notes` field is opaque to Consul but can be used to provide a human-readable
description of the current state of the check. With a script check, the field is
set to any output generated by the script. Similarly, an external process updating
a TTL check via the HTTP interface can set the `notes` value.

Checks may also contain a `token` field to provide an ACL token. This token is
used for any interaction with the catalog for the check, including
[anti-entropy syncs](/docs/internals/anti-entropy.html) and deregistration.

To configure a check, either provide it as a `-config-file` option to the
agent or place it inside the `-config-dir` of the agent. The file must
end in the ".json" extension to be loaded by Consul. Check definitions can
also be updated by sending a `SIGHUP` to the agent. Alternatively, the
check can be registered dynamically using the [HTTP API](/docs/agent/http.html).

## Check Scripts

A check script is generally free to do anything to determine the status
of the check. The only limitations placed are that the exit codes must obey
this convention:

 * Exit code 0 - Check is passing
 * Exit code 1 - Check is warning
 * Any other code - Check is failing

This is the only convention that Consul depends on. Any output of the script
will be captured and stored in the `notes` field so that it can be viewed
by human operators.

## Initial Health Check Status

By default, when checks are registered against a Consul agent, the state is set
immediately to "critical". This is useful to prevent services from being
registered as "passing" and entering the service pool before they are confirmed
to be healthy. In certain cases, it may be desirable to specify the initial
state of a health check. This can be done by specifying the `status` field in a
health check definition, like so:

```javascript
{
  "check": {
    "id": "mem",
    "script": "/bin/check_mem",
    "interval": "10s",
    "status": "passing"
  }
}
```

The above service definition would cause the new "mem" check to be
registered with its initial state set to "passing".

## Service-bound checks

Health checks may optionally be bound to a specific service. This ensures
that the status of the health check will only affect the health status of the
given service instead of the entire node. Service-bound health checks may be
provided by adding a `service_id` field to a check configuration:

```javascript
{
  "check": {
    "id": "web-app",
    "name": "Web App Status",
    "service_id": "web-app",
    "ttl": "30s"
  }
}
```

In the above configuration, if the web-app health check begins failing, it will
only affect the availability of the web-app service. All other services
provided by the node will remain unchanged.

## Multiple Check Definitions

Multiple check definitions can be defined using the `checks` (plural)
key in your configuration file.

```javascript
{
  "checks": [
    {
      "id": "chk1",
      "name": "mem",
      "script": "/bin/check_mem",
      "interval": "5s"
    },
    {
      "id": "chk2",
      "name": "/health",
      "http": "http://localhost:5000/health",
      "interval": "15s"
    },
    {
      "id": "chk3",
      "name": "cpu",
      "script": "/bin/check_cpu",
      "interval": "10s"
    },
    ...
  ]
}
```
