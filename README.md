# Deployment

In this workshop, we'll cover the basics of setting up a barebone deployment pipeline, in support of a green-blue deployment strategy. Concepts covered include green-blue deployment, automatic failover, feature flags, and data migration.

Part 1. [Setup and Overview](README.md) ⬅️  
Part 2. [Configure pipeline and infrastructure](Pipeline.md)  
Part 3. [Implement deployment strategy](Deploy.md)  

## Setup

### Before you get started

Import this as a notebook or clone this repo locally. Also, ensure you [install latest version of docable](https://github.com/ottomatica/docable-notebooks/blob/master/docs/install.md)!

```bash
docable-server import https://github.com/CSC-DevOps/Deployment
```

Install local packages.

``` bash | {type: 'command', failed_when: 'exitCode != 0', stream: true}
npm install
```

Create 2 new virtual machines that will be configured with redis and node.js:
Setup two virtual machines.

``` bash | {type: 'command', failed_when: 'exitCode != 0', stream: true}
bakerx run
```

![blue-green-diagram](img/blue-green.png)

## Blue-green deployment

A blue-green deployment strategy involves two duplicate instances of production infrastructure---one instance receives active traffic while one instance remains dormant and on stand-by. The paired infrastructure can be prepared such that one instance contains experimental or newly deployed code, while the other instance contains a stable baseline. A router or proxy server can be changed to switch to either instance, either manually or automatically. 

This strategy does not necessarily need to be used for the entire infrastructure, but can be used even at the component level. For example, Netflix uses an automatically constructed "red-black" pair of clusters for releasing a change to a microservice---one instance contains the new commit while the other instance contains the previously stable version.  This enables each commit to be automatically released into production---as much as 4000 times a day.

There are several benefits of this using this strategy for deployment. Having two instances can reduce the risks associated with deployment failures, especially during a "big bang" deployment. Rather than figuring out how to rollback the release while possibliy waiting on a new patched environment to be baked, one simply toggles the router. In essence, the strategy can support a "stress-free" preparation of experimental environment. A release can be stablized and patched as needed in production without concerns of releasing during a particular late-night deployment window and hoping for the best.

Downsides also exist. There is a considerable amount of overhead in infrastructure---you are paying at least 2x in terms of infrastructure, including extra capacity used for robustness. Having duplication instances also raises considers with data migration. If you switch versions, how do you ensure consistency and durability of the data? Now you also need to ensure data is mirrored or data migration occurs. Do not forget to overlook certain places data can be misplaced (such as data being queued in a memory-store such as redis).

Finally, when combined with feature flags, new features and problematic changes can be isolated with a simple flag toggle---allowing finer grain control of deployments without requiring a full flip between production instances. For example, consider a release that includes five new features updates, four of which work well while one is broken. A team can now make the decision whether to simply turn off the buggy feature rather than switching everything to the old stable environment. One the other hand, if that buggy feature caused stability issues for the entire system, an environment flip might be the better option.
