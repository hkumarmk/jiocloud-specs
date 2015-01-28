# introduction

We have only considered how to build a single stack from
our rjil deployment tools. The reality is that we need to
be able to deploy other services and manage them separately.

Ideally, we should be able to deploy N number of differnt stacks,
and have each of those stacks updated separately.

# Asumptions

1. It does not make sense to deploy everything together in all cases.

Ex 1: some services need to persist beyond a build:
  - log aggregation services
  - proxy services? (it makes sense to keep it alive between buids to save
    time/network bandwitdh)

Ex: We may want to deploy external services that do not need to be built
    as a part of each stack.

2. We may eventually split services out into their own managed deployment processes.

I could see that eventually, as opposed to updating everything at the same time,
we split them into the following separately managed stages:

* compute
* network
* storage
* contrail plain

Where each is managed by different teams and updated separately (b/c they each have their
own implications to how they can be managed and SLAs)

# solution

* define stacks other than *full* (perhaps full should be renamed openstack-full)
* start with some external dependencies that we need for acceptance (proxy/logstash)

## Code implementation

Assuming we start by having each stack use the same repo snapshot process and each
stack deploys it's own consul server, I'm not really sure if there are any required
code changes.

# Considerations

1. Are there build reliability risks related to not always installing all dependencies
as part of the same stack?

2. Do we want everything to always update at the same time based on the package snapshots?

3. Does this require a redesign of our package repo system?

4. Does each service need to externally propagate it's DNS address?

5. Does each service need it's own consul leader?
