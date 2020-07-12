# Deployment

In this workshop, we'll cover the basics of setting up a barebone deployment pipeline, in support of a green-blue deployment strategy. Concepts covered include green-blue deployment, automatic failover, feature flags, and data migration.

![blue-green-diagram](img/blue-green.png)

### Blue-green deployment

A blue-green deployment strategy involves two duplicate instances of production infrastructure---one instance receives active traffic while one instance remains dormant and on stand-by. The paired infrastructure can be prepared such that one instance contains experimental or newly deployed code, while the other instance contains a stable baseline. A router or proxy server can be changed to switch to either instance, either manually or automatically. 

This strategy does not necessarily need to be used for the entire infrastructure, but can be used even at the component level. For example, Netflix uses an automatically constructed "red-black" pair of clusters for releasing a change to a microservice---one instance contains the new commit while the other instance contains the previously stable version.  This enables each commit to be automatically released into production---as much as 4000 times a day.

There are several benefits of this using this strategy for deployment. Having two instances can reduce the risks associated with deployment failures, especially during a "big bang" deployment. Rather than figuring out how to rollback the release while possibliy waiting on a new patched environment to be baked, one simply toggles the router. In essence, the strategy can support a "stress-free" preparation of experimental environment. A release can be stablized and patched as needed in production without concerns of releasing during a particular late-night deployment window and hoping for the best.

Downsides also exist. There is a considerable amount of overhead in infrastructure---you are paying at least 2x in terms of infrastructure, including extra capacity used for robustness. Having duplication instances also raises considers with data migration. If you switch versions, how do you ensure consistency and durability of the data? Now you also need to ensure data is mirrored or data migration occurs. Do not forget to overlook certain places data can be misplaced (such as data being queued in a memory-store such as redis).

Finally, when combined with feature flags, new features and problematic changes can be isolated with a simple flag toggle---allowing finer grain control of deployments without requiring a full flip between production instances. For example, consider a release that includes five new features updates, four of which work well while one is broken. A team can now make the decision whether to simply turn off the buggy feature rather than switching everything to the old stable environment. One the other hand, if that buggy feature caused stability issues for the entire system, an environment flip might be the better option.

## Workshop

### Before you start

To start with, you'll need some files in this repo to help setup the blue-green infrastructure.

    git clone https://github.com/CSC-DevOps/Deployment.git
    cd Deployment
    npm install

If you do not have the queues image, pull:

    bakerx pull CSC-DevOps/Images#Spring2020 queues

Setup two virtual machines.

    node index setup

### Setting up our Pipeline

We will first setup a basic pipeline that will have two deployment endpoints for our application.

### Initializing our deployment endpoints.

We'll create two endpoints for our deployment, a "green" endpoint for our new production code, and a "blue" endpoint for our baseline.

Inside `GREEN`, `bakerx ssh green`, run:

```bash
mkdir -p meow.io/green.git meow.io/green-www
cd meow.io/green.git
git init --bare
```

Inside `BLUE`, `bakerx ssh blue`, run:

```bash
mkdir -p meow.io/blue.git meow.io/blue-www
cd meow.io/blue.git
git init --bare
```

##### Post-Receive Hook

Like our pipelines workshop, we will use a bare repository and a hook script to receive and checkout changes pushed to our bare repository.

Inside green, `bakerx ssh green`, create a hook file:

```bash
cd meow.io/green.git/hooks/
touch post-receive
chmod +x post-receive
```

Place the following content inside:

    GIT_WORK_TREE=/home/vagrant/meow.io/green-www git checkout -f
    cd /home/vagrant/meow.io/green-www && npm install

Repeat for `BLUE`.

> Note: when repeating for `BLUE` make sure to update the paths inside `post-receive` hook.

### Setting git remotes

On your host computer, clone the [meow.io repo](https://github.com/CSC-DevOps/meow.io), and set the following remotes, using the ssh protocol:

    git clone https://github.com/CSC-DevOps/meow.io
    cd meow.io
    git remote add blue ssh://vagrant@192.168.44.25/home/vagrant/meow.io/blue.git
    git remote add green ssh://vagrant@192.168.44.30/home/vagrant/meow.io/green.git

You can now push changes in the following manner.

    GIT_SSH_COMMAND="ssh -i ~/.bakerx/insecure_private_key" 
    git push green master
    git push blue master

Here, by setting `GIT_SSH_COMMAND`, we are telling git to use our ssh key for connecting to the VM. For Windows 🔽, you will need to modify the command to use `set`, and without quotes.

```bash
set GIT_SSH_COMMAND=ssh -i ~/.bakerx/insecure_private_key
git push green master
git push blue master
```

### Testing deployment

We can go into our blue environment, `bakerx ssh blue`, and start our server:

```
cd meow.io/blue-www
# Create database
node data/init.js
# Start web server
npm start
```

Visit http://192.168.44.25:3000 in your browser to see if meow.io 😻 is running!

Repeat the same for the `GREEN` environment.

## Settting up Infrastructure

Currently, we can deploy changes to our different VMs---however---we have nothing that regulates the control of traffic, nor logic which determines which `TARGET` is active. We will set up our infrastructure to fully handle a deployment, including automatic failover.

You may want to setup your terminals to help you distinguish your `GREEN` and `BLUE` environments, as follows:

![setup](img/meow-deploy.png)

### Task 1: Configure Proxy

We will be using our host environment coordinate our production environment with a proxy service. 
Complete the proxy service by adding a redirect from `localhost:3080` to our `TARGET` production endpoint.

```js
proxy.web( req, res, {target: self.TARGET } );
```
To activate, run `node index.js serve`. Visiting http://localhost:3080 should redirect you to the `GREEN` production environment.


### Task 2: Configure process supervisor

We want our environment to automatically restart the web server when a push is made.

Inside the blue environment, `bakerx ssh blue`, install forever:

```
sudo npm install forever -g
```

We will then append the following to our post-receive hook:

```bash
forever stopall
forever -w start ./bin/www
```

Repeat for the green environment, `bakerx ssh green`.

##### Trigger hooks to run supervisor

We are going to amend our last commit, to enable us to push and trigger our post-receive hooks.

```bash
git commit --amend --no-edit
git push blue master -f
git push green master -f
```

Our forever process should now running our web server without us needing to manually stop or start it. You can confirm with `forever list` on each server.

### Task 3: Add automatic failover

In case a bad commit is pushed to our green environment, we want a way to automatically direct traffic back to our stable `BLUE` environment.

We will accomplish this by adding a health monitor, which checks every 5 seconds if the `GREEN` environment has any failure. If failure does occur, then it will automatically switch the `TARGET` to the `BLUE` environment.

Update the `healthCheck()` function to perform the switch.

##### Commit bad commit, trigger failover, revert bad change

We will introduce a bad commit to the `GREEN` environment which should trigger our failure over.

Modify the index route in meow.io, to explicitly fail: `res.status(500).render(...`. Then commit and push the change:

```
$ git add index.js
$ git commit -m "bad commit"
$ git push green master
```

You should observe your failover being triggered---traffic should now be served from the `BLUE` environment.

We can use this chance to patch production and revert our bad commit:
```
$ git revert HEAD
[master 5edde92] Revert "bad commit"
 1 file changed, 1 insertion(+), 1 deletion(-)

$ git push green master
```

Restart the `node index.js serve` process to restore traffic to the `GREEN` environment.

### Task 4: Feature Flag

Notice that meow.io is not currently displaying any recent images. Set the value `RECENT=ON` inside the redis server on the `BLUE` environment.

![off](img/meow.io-off.png)

You should be able to see the feature turned on:

![on](img/meow.io-flag.png)

A feature flag can be a powerful tool to assist with deployment. It can be used to fence off new features or quickly turn off buggy features without needing to rollback a commit.

### Task 5: Extra---Sync/migrate the database

Our infrastructure handles a lot now, but there is one detail missing. If a switch occurs while the `GREEN` environment received data, that data would be lost when switching back to the `BLUE` environment.

We have three possible strategies we can add to handle this which making the switch over.

**Strategy 1**: Copy database file.

This strategy will dump and then move over the whole sqlite database.
```bash
sqlite3 "/home/vagrant/green-www/data/meowio.db" ".backup /tmp/dbbackup.sqlite"
rsync vagrant@BLUE:"/tmp/dbbackup.sqlite" "/home/vagrant/blue-www/data/meowio.db"
```

While simple, the strategy might harm service availability and data durability, as the whole dataset must be copied before serving traffic, and there may be data corruption.

**Strategy 2**: Migrate missing rows table by table.

Missing rows can be appended to the `BLUE` database. This can be done before or after making switching. This strategy could effect consistency of the data.

**Strategy 3**: Mirror data.

Having the `GREEN` database mirror the data to the `BLUE` database allows the system to move data over during normal operation, and not during an emergency switch. Unfortunately, sqlite does not support mirroring, however, the code could be modified to mimic mirroring, or a real database system could be used instead.

