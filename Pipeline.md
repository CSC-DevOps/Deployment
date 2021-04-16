<!--
targets:
    - type: bakerx
      name: green
    - type: bakerx
      name: blue
    - type: local
      name: local
-->


# Configure pipeline and infrastructure

Part 1. [Setup and Overview](README.md)   
Part 2. [Configure pipeline and infrastructure](Pipeline.md) ⬅️  
Part 3. [Implement deployment strategy](Deploy.md)  

### Setting up our Pipeline

We will first setup a basic pipeline that will have two deployment endpoints for our application.

### Initializing our deployment endpoints.

We'll create two endpoints for our deployment, a "green" endpoint for our new production code, and a "blue" endpoint for our baseline.

Inside `GREEN`, `bakerx ssh green`, run:

```bash
mkdir -p meow.io/green.git meow.io/www
cd meow.io/green.git
git init --bare
```

``` | {type: 'terminal', 'background-color': '#007036', target: 'green'} 
```

Inside `BLUE`, `bakerx ssh blue`, run:

```bash
mkdir -p meow.io/blue.git meow.io/www
cd meow.io/blue.git
git init --bare
```

``` | {type: 'terminal', 'background-color': '#003670', target: 'blue'} 
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

```bash
GIT_WORK_TREE=/home/vagrant/meow.io/www git checkout -f
cd /home/vagrant/meow.io/www && npm install
[ ! -f "data/meowio.db" ] && node data/init.js
npm stop
npm start
```                             

Repeat for `BLUE`.

### Setting git remotes

On your host computer, clone the [meow.io repo](https://github.com/CSC-DevOps/meow.io), and set the following remotes, using the ssh protocol:

    git clone https://github.com/CSC-DevOps/meow.io
    cd meow.io
    git remote add blue ssh://vagrant@192.168.44.25/home/vagrant/meow.io/blue.git
    git remote add green ssh://vagrant@192.168.44.30/home/vagrant/meow.io/green.git


``` | {type: 'terminal', target: 'local'} 
```


You can now push changes in the following manner.

    export GIT_SSH_COMMAND="ssh -i ~/.bakerx/insecure_private_key -o StrictHostKeyChecking=no" 
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