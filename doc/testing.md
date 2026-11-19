# Allium Development Validation

This is an overview of test strategy for the initial development of Allium.

Of course, some of this is aspirational, but efforts and agents should attempt
to ensure we adhere to these goals.

## Unit Testing

All code will have unit tests with code coverage metrics.

## Simulation

We plan to use https://github.com/dtalayco/sonic-lab (or another simulation 
environment) to deploy various topologies for first level deployment
validation. This can be used to validate the interaction between the
controller and individual switch instances. It can also validate much of
the forwarding functionality across different topologies.

## Resiliency

Figure out how to test failover and other controller resilience test cases.

## Realistic Hardware Deployments

Reflecting on the challenge of deploying the controller across
hardware deployments of interesting complexity reveals the ambition of
this project. We defer such considerations until we have developed the
controller sufficiently to justify finding such resources against
which to deploy and test the system.

## Real Performance

As with the previous section, we note the importance of performance
measurement and regression testing, but defer further development outside
of simulation cases.

## Acceptance Testing

Code commmits will be subject to regression testing covering unit tests
and some simulation deployments. We will use GitHub actions to deploy
a set of tests that are run against pull requests.

These test suites will be a subset of those available for developers to
deploy themselves for development testing.

