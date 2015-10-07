CF-Abacus
===

The Abacus usage metering and aggregation service.

[![Build Status](https://travis-ci.org/cloudfoundry-incubator/cf-abacus.svg)](https://travis-ci.org/cloudfoundry-incubator/cf-abacus) [![Coverage Status](https://coveralls.io/repos/cloudfoundry-incubator/cf-abacus/badge.svg?branch=master&service=github)](https://coveralls.io/github/cloudfoundry-incubator/cf-abacus?branch=master) [![Slack Team](https://abacusdev-slack.mybluemix.net/badge.svg)](https://abacusdev-slack.mybluemix.net/) [![Gitter Chat](https://img.shields.io/badge/gitter-join%20chat-blue.svg)](https://gitter.im/cloudfoundry-incubator/cf-abacus?utm\_source=badge) [![IRC Chat](https://img.shields.io/badge/irc-%23abacusdev-blue.svg)](http://webchat.freenode.net?channels=%23abacusdev)

Overview
---

Abacus provides usage metering and aggregation for [Cloud Foundry (CF)](https://www.cloudfoundry.org) services. It is implemented as a set of REST micro-services that collect usage data, apply metering formulas, and aggregate usage at several levels within a Cloud Foundry organization.

Abacus provides a REST API allowing Cloud service providers to submit usage data, and a REST API allowing usage dashboards, and billing systems to retrieve usage reports.

Abacus is implemented in Node.js and the different micro-services can run as CF apps.

The Abacus REST API is described in [doc/api.md](doc/api.md).

Frequently Asked Questions (FAQs)
---

1. How to get Usage Data from CF?

  We are currently building a prototype for reporting CF apps usage data to Abacus. You can have a look at the bridge source.
  
  What we currently do is to:
  * obtain oauth token 
  * list app usage events
  * report linux-container metrics   
  
  For now we basically report the memory used.
  
  The same approach can be applied to service usage events. However I'm not aware of any useful metrics (besides the number of instances) present in the service usage events. This is due to the fact that CF does not (and should not) have an idea what the service actually does.
  
  Instead the service providers can report the metrics they want to charge for to Abacus. This can happen in the service broker itself or in dedicated bridge that takes care to fetch the metrics from the broker and report the usage to Abacus.

2. Is there an API to cancel or correct entries?

  Our current strategy is to have the service or runtime provider submit compensating usage (e.g. if that service provider thinks it has submitted too much GB hours for example, he can always compensate that with a negative/credit usage later). If there's a mistake, it's compensated later, we don't rewrite history, for many reasons including auditing, potentially legal reasons etc...
  
  The other thing to consider is that even if there was a way to rewrite history, that wouldn't handle the cases where the mistaken usage would have already been shown to the customer (as customers are typically able to see their usage in close to real time) or, even more difficult... the cases where that usage would have already been billed to that customer (e.g. the mistake happened on 08/31 23:30pm and only got discovered on 09/01 after a bill was produced and sent to the customer...)

3. Can one configure CF-Abacus for a trial period?

  Abacus allows you to completely control how usage and costs are accumulated over time, and aggregated at various levels inside an organization using Javascript functions that you can define yourself for each resource, so yes you should be able to implement that kind of logic.
  
  We are actually in the middle of implementing time windows (sec, min, hour, day, month, year) for usage accumulation so when that's in place it'll be even easier to do it.
  
  Another thing to consider is that Abacus doesn't do billing/invoicing. We meter, aggregate and rate usage to produce usage and charge summary reports that can then be fed to a billing system. Your particular billing system can eventually decide to apply discounts or 'just make it free for that customer' for example on top of the usage and rating data that Abacus provided.

4. How does the report generated by the demo relate to the entered usage and rate data?

  In a nutschell:
  
  * a resource provider controls how usage is metered, accumulated, aggregated, rated, summarized and charged through a set of Javascript functions (meter, accumulate, aggregate, rate, summarize, charge) that can be configured in a resource configuration JSON doc (see examples [1][2]); these functions are called to process submitted usage as it flows through the Abacus meter, accumulator, aggregator and reporting services;
  
  * the resource provider submits usage docs containing usage measures (e.g. number of API calls, or bytes of storage or memory);
  
  * the resource provider's meter() function is called, given the usage measures, and is responsible for converting them to the metrics relevant for the particular resource (e.g. thousands of API calls, GB of storage, MB of memory);
  
  * the provider's accumulate() function is called, responsible for accumulating (usually a simple sum) the particular metric over time (per sec, min, hour, day, month etc);
  
  * the provider's aggregate() function is called next, responsible for aggregating (usually a sum as well) the particular metric at different levels under an organization;
  
  * next, the provider's rate() function is called, responsible for computing the cost of the metric given the usage and the price of the particular metric;
  
  * finally the summarize() and charge() functions are called to compute the final usage summary and corresponding charges, at the time the usage summary report gets produced.
  
  * one final note, usage measures and metrics can be expressed as simple numbers (e.g. a decimal number representing a number of API calls) or compound data structures for more sophisticated usage metering schemes (e.g the ability to represent something like 'already consumed 153GB of memory and now consuming 3GB since 9:32am today'); the provider's meter, accumulate, aggregate etc functions are then free to interpret the data structure representing the metric however they want.
  
  [1] https://github.com/cloudfoundry-incubator/cf-abacus/blob/master/lib/config/resource/src/resources/storage.js
  [2] https://github.com/cloudfoundry-incubator/cf-abacus/blob/master/lib/config/resource/src/resources/container.js

5. Can we set different prices for different customers?

  The pricing is currently loaded from a simple JSON configuration file, but the intent is to convert that to a service that an integrator of Abacus can implement to control what price is used for a particular resource, plan, organization, time (as prices evolve over time), country, currency etc.
  
  That service will need to implement a REST API that Abacus will call to get the price to use to rate the usage. So, yes, with that you'll be able to return a different pricing per customer (i.e. per organization).

Building
---

Abacus requires Npm >= 2.10.1 and Node.js >= 0.10.36 or io.js >= 2.3.0.

```sh
cd cf-abacus

# Bootstrap the build environment, run Babel on the Javascript sources,
# install the Node.js module dependencies and run the tests
npm run build
```

Testing
---

```sh
cd cf-abacus

# Run eslint on the Abacus modules
npm run lint

# Run the tests
npm test
```

Deploying to Cloud Foundry
---

Abacus runs as a set of applications deployed to Cloud Foundry. Each application is configured to run in multiple instances for availability and performance. Service usage data is stored in CouchDB databases.

This diagram shows the main Abacus apps and their role in the processing of usage data.

![Abacus flow diagram](doc/flow.png)

The following steps assume a local Cloud Foundry deployment created using [Bosh-lite](https://github.com/cloudfoundry/bosh-lite) and running on the default local IP 10.244.0.34 assigned to that deployment. Please adjust to your particular Cloud Foundry deployment environment.

```sh
cd cf-abacus

# Point the CF CLI to your local Cloud Foundry deployment
cf api --skip-ssl-validation https://api.10.244.0.34.xip.io
cf login -o <your organization> -s <your space>

# Create a CF security group for the Abacus apps
cat >abacus_group.json <<EOF
[
  {
    "destination": "10.0.0.0-10.255.255.255",
    "protocol": "all"
  }
]
EOF
cf create-security-group abacus abacus_group.json
cf bind-security-group abacus <your organization> <your space>

# Run cf push on the Abacus apps to deploy them to Cloud Foundry
npm run cfpush

# Check the state of the Abacus apps
cf apps

# You should see something like this
Getting apps in org <your organization> / space <your space>...
OK

name                       requested state   instances   memory   disk   urls   
abacus-usage-collector     started           1/1         512M     512M   abacus-usage-collector.10.244.0.34.xip.io   
abacus-usage-meter         started           1/1         512M     512M   abacus-usage-meter.10.244.0.34.xip.io 
abacus-usage-accumulator   started           1/1         512M     512M   abacus-usage-accumulator.10.244.0.34.xip.io   
abacus-usage-aggregator    started           1/1         512M     512M   abacus-usage-aggregator.10.244.0.34.xip.io   
abacus-usage-reporting     started           1/1         512M     512M   abacus-usage-reporting.10.244.0.34.xip.io   
abacus-dbserver            started           1/1         1G       512M   abacus-dbserver.10.244.0.34.xip.io   
```

Running the demo on Cloud Foundry
---

The Abacus demo runs a simple test program that simulates the submission of usage by a Cloud service provider, then gets a daily report for the usage aggregated within a Cloud Foundry organization.

The demo data is stored in a small in-memory [PouchDB](http://pouchdb.com) test database so the demo is self-contained and you don't need to set up a real CouchDB database just to run it.

Once the Abacus apps are running on your Cloud Foundry deployment, do this:

```sh
cd cf-abacus

# Run the demo script
npm run demo -- \
  --collector https://abacus-usage-collector.10.244.0.34.xip.io \
  --reporting https://abacus-usage-reporting.10.244.0.34.xip.io

# You should see usage being submitted and a usage report for the demo organization

```

Running Abacus on localhost
---

The Abacus apps can also run on your local host in a shell environment outside of Cloud Foundry, like this:

```sh
cd cf-abacus

# Start the Abacus apps
npm start

# Wait a bit until all the apps have started

# Run the demo script
npm run demo

# Stop everything
npm stop
```

Meter Cloud Foundry App Usage
---

Abacus comes with a [bridge](lib/cf/bridge) that acts as a Service Provider. It reads Cloud Foundry's [App Usage Events](http://apidocs.cloudfoundry.org/runtime-passed/app_usage_events/list_all_app_usage_events.html) and reports usage to the `abacus-usage-collector`. In the end it enables you to see usage reports for your Cloud Foundry instance. In order to start the bridge follow its [readme](lib/cf/bridge/README.md) 

Layout
---

The Abacus source tree is organized as follows:

```sh

bin/ - Start, stop, demo and cf push scripts 

demo/ - Demo apps

    client - demo program that posts usage and gets a report

doc/ - API documentation

lib/ - Abacus modules

    metering/ - Metering services

        collector - receives and collects service usage data
        meter     - applies metering formulas to usage data

    aggregation/ - Aggregation services

        accumulator - accumulates usage over time
        aggregator  - aggregates usage within an organization
        reporting   - returns usage reports

    config/ - Usage formula configuration
    
    rating/ - Rating services
    
        rate - applies pricing formulas to usage

    utils/ - Utility modules used by the above

    stubs/ - Test stubs for provisioning and account services

test/ - End to end tests

    perf/ - Performance tests

tools/ - Build tools

etc/ - Misc build scripts

```

People
---

[List of all contributors](https://github.com/cloudfoundry-incubator/cf-abacus/graphs/contributors)

Development
---

The Abacus project has a tree structure that can be seen above. It is a module consisting of submodules. These submodules are located under the `lib` directory. When developing locally you often need to make changes to one submodule only, build and run the tests, rather than rebuilding all submodules.

Let's take the metering collector module as an example. First you have to build all modules:

```
cd cf-abacus
npm run build
```

Then you have to install your submodule's dependencies

```
cd cf-abacus/lib/metering/collector
npm install
```

At this point your dev cycle boils down to:

```
cd cf-abacus/lib/metering/collector
npm run babel && npm run lint && npm test
```

If you want to run the collector app you can run:

```
cd cf-abacus/lib/metering/collector
npm start
```

License
---

  [Apache License 2.0](LICENSE)

