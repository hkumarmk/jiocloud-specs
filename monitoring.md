# introduction

Our monitoring requirements are driven by two factors:

1. scale

2. repeatability of monitoring processes as a part of the
   developmemt process

# Asumptions

1. A traditional monitoring tool cannot scale b/c it
   tries to maintain too much centralized state (or perform
   too many centralized tasks).

2. Centralized monitoring makes it too hard to have monitoring
   serve as way to test code as you develop.

3. As much logic as possible should be performed on the individual
   nodes so that the load of testing is distributed as much as possible.

4. The volume of results stored should be limited as much as possible.

# solution

Use etcd and a custom framework to improvement the performance of
monitoring by reducing the need of a central service to do work.

## High Level

Run all monitoring tasks on the local systems.

The machine should be updated in etcd _only_ in the case of a failure.
To indicate as failed, it should be added to the _failures_ directory.
The results of the failure should also be pushed to swift, and a link
to the swift results should be associated with the failure/<node> directory.

## Code implementation

the project should be a simple plugin that does the following:

* Look for specified plugins in a special directory

/var/lib/<tool\_name>/<plugin\_name>/<tests>

* For each plugin, run the tests using that plugin

* It may make sense to use a unit test framework to display results

* Capture the end result of which tests passed/failed

* Capture the end result of whether all tests passed

* If any tests fail, write test output to swift, and mark
  node as failed in etcd

* When things don't fail, write them locally, and occassionally sync
  passed results to swift.

## metrics

store metrics locally. Offer a local http connection to view metrics for
each server.

## Things to look into

- Can we just use serverspec for this?
- Can we reuse nagios plugins

## Unknowns

What about aggregated tests? (things that have to know if a certain condition exists
across multiple hosts)

Be able to stop notifications (send a signal in hubot)
