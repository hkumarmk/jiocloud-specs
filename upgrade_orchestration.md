# Introduction

Upgrading our cloud systems will become one of the most common operations that
we perform. It needs to be based on a design that can ensure continuous
automated upgrades that are generally flawless, with tooling in place to make
decisions about what do it in case of failure.

# Current Solution

The current solution provides the capability to upgrade via the most primitive
mechanism possible. A single desired version is injected into our key/value
store. Each agent is constantly polling that key while asking the question:

  - should I upgrade

It answers this question by comparing it's local version to the version published
in the key/value store. If the answer is yes, it proceeds to upgrade.

In this approach, and upgrade either rolls completely through all hosts, or it
fails. In the case of failure, manual intervention is required to root-cause
the failure and resolve the issue.

# Issues with current Solution

While the current solution does support upgrades and is implemented in such a
way that each individual machine is capable of upgrading itself, it does not
take into account the need for having more control over the orchestration of
upgrades.

This approach does not take into account a few considerations that will be required
for our future upgrades of Openstack.
1. upgrades of roles may depend on other roles
2. certain types of operations may need to be upgraded in isolation
3. rolling upgrades are required to help facilitate recovery of failure cases

It also does not adequately account for the fact that an upgrade may fail, and
that failure may effect the uptime of our cloud environment. In this case,
we need well defined operational procedures and tooling that we can use to
recover from these failures as quickly as possible.

# Proposed Solution/Implementation

The current proposed solution should be viewed as an incremental approach that
will help incremental drive us towards our goals.

## orchestrating upgrades

Currently, each agent asks the question: "Should I upgrade". This proposal
looks to add additional code to python-jiocloud so that this question
takes into account the above mentioned considerations.

We should expand this, so that it is determined by the following:

- am I running the desired version?
- is it my roles turn?
- is it my sets turn (for rolling upgrades)?

### role dependencies

In some cases, roles should only perform an upgrade once another role
has completed it's upgrade. To give a few examples:

1. no one should upgrade while a db schema migration occurs.
2. compute hosts should only upgrade once everything else is finished.

To support this feature, two things need to be implemented:

1. A way to specify the dependencies between roles (does this belong in the
  layout?)
2. A way for each host to determine if he can upgrade based on what roles have
   completed upgrading to what versions.

I am definitely less certain about this implementation, but here's a stab:

Each role specified in the layout could support the depends key whose value is set
to the roles that a role will wait for when determining if it should upgrade.

During the build process, this dependencies could be written into the k/v store.

When each node is trying to determine if he should upgrade he compares his role to
the list of dependant roles to determine if he should upgrade.

- is my version not current?
  - does my role have dependencies?
    - are all nodes running those roles at the current version?

### creating sets

All machines can be broken up into N sets by using integer remainders.

For example: if the key NUMSETS is set to 3, then each host should perform
remainder division on its self and put itself into either group 1,2,3.

As long as we keep track of which hosts are running what versions of a service
(this is currently implemented as a key per host, but will likely be moved to
consul as a service tag), we can query this to determine if we should run or not:

for example:

if my hostname is nova-controller2 and NUMSETS is set to 3, I can use 2%3
to determine that I am in set 2. In this case should I upgrade would ask:

- Am I not running the correct version?
  - Are all nova-controllers in set 1 running the correct version?
    - I should upgrade

## dealing with failures

Upgrades will fail. It will happen, the best that we can do it to provide
tooling that can allow operators to quickly perform analysis and make a
decision about what to do based on a well defined set of tools.

By default if a set or role fails to upgrade during a specified amount of time,
then the following things should happen:

1. upgrade should fail
2. upgrade should stop
3. alerts should go out

# Alternative Solutions (Pros/Cons)

Did not consider alternatives.

# Deviations from Design guidelines

Requires storing enough state to make decisions about upgrades.

# Considerations

# Unknowns

If we consider nodes as a table with rows being roles and columns sets, we can
imagine how an upgrade might roll through the table. It can either move horizontally
or vertically, or use some combination of both to work its way through the cells.

I proposed that upgrade machines one column at a time (if possible) allows for
much greater possibilities to recovery.

It you only upgrade 10% of your hosts to a new version, and something goes
wrong, it should be pretty easy to stop the rest of the build process,
isolate those nodes, and just rebuild them at the previous version while
you debug the issue.

