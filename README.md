# Cloud Foundry Services

This project is a bosh release containing services that work with Cloud Foundry v2.

## Installation

```
$ ./update
$ printf '%s\n%s\n' '---' 'dev_name: cf-services' > config/dev.yml
$ bosh create release --force
$ bosh -n upload release
```

Your bosh will how have this bosh release available with the name `cf-services`

```
$ bosh releases                  

+-------------+-----------+-------------+
| Name        | Versions  | Commit Hash |
+-------------+-----------+-------------+
...
| cf-services | 0.1-dev   | 7e5dbbf3+   |
+-------------+-----------+-------------+
```

## Templating

This project has a series of templates that are parsed and combined by a tool named spiff. The manifest sections shown in this README
are a result of this parsing. BOSH will use the manifest with the sections like those in this readme.

## Usage

You can now include one or more of these services (gateway and nodes) in your Cloud Foundry deployment file. In the examples below, MySQL will be deployed.

First, add the release to the `releases` section:

``` yaml
---
name: cf
director_uuid: UUID
releases:
- name: appcloud
  version: 131.7-dev
- name: cf-services
  version: 0.1-dev
...
```

Next, describe two new jobs _after all other jobs_. One for the `mysql_gateway` job template and another for the first plan that will be offered to users, via `mysql_node` job template.

``` yaml
jobs:
...
- name: mysql_gateway
  template:
  - mysql_gateway
  release: cf-services
  resource_pool: small
  instances: 1
  networks:
  - name: default
    default:
    - dns
    - gateway
  properties:
    uaa_endpoint: http://uaa.aws.drniccloud.com
    uaa_client_id: cf
    uaa_client_auth_credentials:
      username: services
      password: c1oudc0w
    cc_api_version: v2
    token: c1oudc0wc1oudc0w
    default_plan: default
    supported_versions:
    - '5.5'
    version_aliases:
      current: '5.5'
- name: mysql_node
  template:
  - mysql_node
  release: cf-services
  instances: 1
  resource_pool: small
  persistent_disk: 4096
  networks:
  - name: default
    default:
    - dns
    - gateway
  properties:
    mysql_node:
      plan: default
      supported_versions:
      - '5.5'
      password: c1oudc0wc1oudc0w
```

Note: each job above specifies `release: cf-services`.

The configurable aspects above are the amount of `persistent_disk` (set to 4G above) for the job to share amongst the provisioned databases, and the plan to be offered by the initial provisioned `mysql_node` job (`plan: default`).

Also, the `uaa_client_auth_credentials` must match one of those within the UAA `properties` configuration.

Next, ensure that any other Cloud Foundry jobs within the deployment file explicitly reference their bosh release.

For example, if you have a collocated job `core` containing many jobs of the `appcloud` release (see `bosh releases` output for its name within your bosh), then include `release: appcloud` in each of your original jobs:

```
jobs:
- name: core
  template:
  - syslog_aggregator
  - nats
  - postgres
  - health_manager_next
  - collector
  - debian_nfs_server
  - login
  release: appcloud
  instances: 1
  resource_pool: medium
  persistent_disk: 8192
  networks:
  - name: default
    default:
    - dns
    - gateway
  properties:
    db: databases
...
```

Next, specify the available plans. The service gateway needs to know about all available plans and it educates the Cloud Controller about them once its running. Each job using the `mysql_node` job template needs to state which specific plan it will run. Above, we assumed there was a plan called `default`.

Let's now describe this `default` plan, and the authentication credentials for the gateway. This configuration is within the `properties` section of your deployment file:

``` yaml
properties:
...
  service_plans:
    mysql:
      default:
        unique_id: 'some-random-guid-or-string'
        description: "Default MySQL plan."
        extra: '{"cost":0.0,"bullets":[{"content":"100 MB storage"},{"content":"10 connections per DB"}]}'
        configuration:
          capacity: 20
          max_db_size: 100
          innodb_buffer_pool_size: 512
          max_clients: 10
          max_connections: 100
```

The service plan `default` is described from the `properties.service_plans.mysql.default` key name above.

### Deploy and enable services

There are two remaining steps to deploy and use the services:

* deploy the updated bosh deployment file above (using `bosh` CLI)
* register the service (using `cf` CLI)

First, run `bosh deploy` to compile the additional packages and provision/apply the new job VMs:

```
$ bosh deployment deployments/cf.yml
$ bosh deploy
```

The service gateway and initial service node are now running and will appear to your Cloud Foundry users (BUT, please ahead, do not stop yet!).

```
$ cf services --marketplace
```

Finally, install the `admin-cf-plugin` plugin for `cf` CLI and register the new service gateway:

```
$ gem install admin-cf-plugin
$ cf create-service-auth-token mysql core --token c1oudc0wc1oudc0w
```


## Troubleshooting

If you do not create the service token via the `admin-cf-plugin` plugin above, your users will not be able to create services.

```
$ cf create-service
...
```
