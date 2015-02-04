# intro

This repository holds specifications that propose and discuss solutions and
features. This will ensure that we reach design concensus on higher level
features before implementation begins.

It will allow some people to get started by implementing written specifications
as well as others to get started designing features through specifications.

It is intended as a way to scale the team of contributors.

# Starting a new spec

1. Copy the example.md file to a file for your specification.
2. Fill in relevant data.
3. Submit a pull request
4. Collaborate until merged.
5. Implement feature.

# Collaborating on specification (and getting specifications accepted)

Specification review, collaboration, and acceptance will occur via github
pull requests. The hope is that we can use pull request comments to discuss
specifications and merge them when they are accepted.

This repo is a bit of an experimentation is itself, so it will be interesting to
see what happens.

# Architecture Design Guiding Principals

Since the intention of this repo is to have higher level architectural conversations
upfront, it is importland to ensure that these design approached conform to our
design principals (WIP).

## Automate everything

Humans make too many mistakes to have them perform any manual repetitive tasks. Automating tasks also results in fewer mistakes and allows tasks to scale.

## Hands off

No one should ever make any adhoc modification to indivual cloud systems. Changes should all go through the build pipeline.
Adhoc changes risk that certain systems can wind up in an unanticipated state.
Machines with inconsistent states do not confirm with the expectations used to validate builds via the pipeline.

## Keep up-to-date with master

Stay as close to master as possible - The closer you are to master, the less code you need to deploy to stay up-to-date. This should limit the likelihood
that any change set will lead to a failure.

## Failures come from a lack of tests

Tests are the gateway for code getting deployed. We have to fully trust our automated tests in order to validate that our services are running correctly.

## Design for massive scale

Design decisions are made assuming that we eventually have to reach 10s or 100s of thousands of managed systems. Because of this, we intend to limit the number of centralized services as much as possible and expect each of the individual servers of our cloud to take on as much of the processing related to their own management as possible.

## Limit environemnt inconsistencies

Make the deployments as similar as possible for each environment.

