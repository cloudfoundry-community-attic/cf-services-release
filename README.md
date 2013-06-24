# Cloud Foundry Services

This project is a bosh release containing services that work with Cloud Foundry v2.

## Installation

```
$ ./update
$ echo "---\ndev_name: cf-services" > config/dev.yml
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

## Usage

You can now include one or more of these services (gateway and nodes) in your Cloud Foundry deployment file. In the examples below, PostgreSQL will be deployed.

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
```

Next, describe two new jobs _after all other jobs_. One for the `postgresql_gateway` job template and another for the first plan that will be offered to users, via `postgresql_node` job template.

``` yaml
jobs:
...
- name: service_gateways
  template:
  - postgresql_gateway
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
- name: postgresql_service_node
  template:
  - postgresql_node
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
    plan: default
```

Note: each job above specifies `release: cf-service`.

The configurable aspects above are the amount of `persistent_disk` (set to 4G above) for the job to share amongst the provisioned databases, and the plan to be offered by the initial provisioned `postgresql_service_node` job (`plan: default`).

Also, the `uaa_client_auth_credentials` must match one of those within the UAA `properties` configuration.

Next, ensure that any other Cloud Foundry jobs within the deployment file explicitly reference their bosh release.

For example, if you have a colocated job `core` containing many jobs of the `appcloud` release (see `bosh releases` output for its name within your bosh), then include `release: appcloud` in each of your original jobs:

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

Next, specify the available plans. The service gateway needs to know about all available plans and it educates the Cloud Controller about them once its running. Each job using the `postgresql_node` job template needs to state which specific plan it will run. Above, we assumed there was a plan called `default`.

Let's now describe this `default` plan, and the authentication credentials for the gateway. This configuration is within the `properties` section of your deployment file:

``` yaml
properties:
...
  service_plans:
    postgresql:
      default:
        job_management:
          high_water: 1400
          low_water: 100
        configuration:
          lifecycle:
            enable: false
          warden:
            enable: false
  postgresql_node:
    supported_versions:
    - '9.1'
    default_version: '9.1'
    password: c1oudc0wc1oudc0w
  postgresql_gateway:
    cc_api_version: v2
    token: c1oudc0wc1oudc0w
    default_plan: default
    supported_versions:
    - '9.1'
    version_aliases:
      current: '9.1'
```

The service plan `default` is described from the `properties.service_plans.postgresql.default` key name above.

### Deploy and enable services

There are two remaining steps to deploy and use the services:

* deploy the update bosh deployment file above (using `bosh` CLI)
* register the service (using `cf` CLI)

First, run `bosh deploy` to compile the additional packages and provision/apply the new job VMs:

```
$ bosh deployment deployments/cf.yml
$ bosh deploy
```

The service gateway and initial service node are now running and will appear to your Cloud Foundry users (BUT, please ahead, do not stop yet!).

```
$ cf info --services
```

Finally, install the `admin-cf-plugin` plugin for `cf` CLI and register the new service gateway:

```
$ gem install admin-cf-plugin
$ cf create-service-auth-token postgresql core --token c1oudc0wc1oudc0w
```


## Troubleshooting

If you do not create the service token via the `admin-cf-plugin` plugin above, your users will not be able to create services.

```
$ cf create-service
...
```