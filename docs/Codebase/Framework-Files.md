There are a few files that are necessary to implement a framework into the Framework Benchmarks. All of these files will vary because they're all specific and unique to each framework. 

| File | Summary |
|:---- |:------- |
[Install File](#install-file) | Installs the framework and all of the dependencies for the framework (if not already installed).
[Setup File](#setup-file) | Configures the framework to the correct database hosts, package framework code, and starts the framework.
[Benchmark Config File](#benchmark-config-file) | Defines test instructions and metadata for the framework benchmarks program.

# Install File

The `install.sh` file for each framework starts the bash process which will 
install that framework. Typically, the first thing done is to call `fw_depends` 
to run installations for any necessary software that TFB has already 
created installation scripts for. TFB provides a reasonably wide range of 
core software, so your `install.sh` may only need to call `fw_depends` and 
exit. Note: `fw_depends` does not guarantee dependency installation, so 
list software in the proper order e.g. if `foo` depends on `bar`
use `fw_depends bar foo`.

Here are some example `install.sh` files

```bash
#!/bin/bash

# My framework only needs nodejs
fw_depends nodejs
```

```bash
#!/bin/bash

# My framework needs nodejs and mono and go
fw_depends nodejs mono go
```

```bash
#!/bin/bash

# My framework needs nodejs
fw_depends nodejs

# ...and some other software that there is no installer script for.
# Note: Use IROOT variable to put software in the right folder. 
#       You can also use FWROOT to refer to the project root, or 
#       TROOT to refer to the root of your framework
# Please see guidelines on writing installation scripts
wget mystuff.tar.gz -O mystuff.tar.gz
untar mystuff.tar.gz
cd mystuff
make --prefix=$IROOT && sudo make install
```

To see what TFB provides installations for, look in `toolset/setup/linux`
in the folders `frameworks`, `languages`, `systools`, and `webservers`. 
You should pass the filename, without the ".sh" extension, to fw_depends. 
Here is a listing as of July 2014: 

```bash
$ ls frameworks                                                                
grails.sh  nawak.sh  play1.sh  siena.sh     vertx.sh  yesod.sh
jester.sh  onion.sh  play2.sh  treefrog.sh  wt.sh
$ ls languages
composer.sh  erlang.sh   hhvm.sh   mono.sh    perl.sh     pypy.sh     racket.sh   urweb.sh
dart.sh      go.sh       java.sh   nimrod.sh  python2.sh  ringojs.sh  xsp.sh
elixir.sh    haskell.sh  jruby.sh  nodejs.sh  php.sh      python3.sh  ruby.sh 
$ ls systools
leiningen.sh  maven.sh
$ ls webservers
lapis.sh  mongrel2.sh  nginx.sh  openresty.sh  resin.sh  weber.sh  zeromq.sh
```

# Setup File

The setup file is responsible for starting and stopping the test. This script is responsible for (among other things):

* Modifying the framework's configuration to point to the correct database host
* Compiling and/or packaging the code (if impossible to do in `install.sh`)
* Starting the server
* Stopping the server

The setup file is a python script that contains a start() and a stop() function.  
The start function should build the source, make any necessary changes to the framework's 
configuration, and then start the server. The stop function should shutdown the server, 
including all sub-processes as applicable.

#### Configuring database connectivity in start()

By convention, the configuration files used by a framework should specify the database 
server as `localhost` so that developing tests in a single-machine environment can be 
done in an ad hoc fashion, without using the benchmark scripts.

When running a benchmark script, the script needs to modify each framework's configuration
so that the framework connects to a database host provided as a command line argument. 
In order to do this, use `setup_util.replace_text()` to make modifications prior to 
starting the server.

For example:

```python
setup_util.replace_text("wicket/src/main/webapp/WEB-INF/resin-web.xml", "mysql:\/\/.*:3306", "mysql://" + args.database_host + ":3306")
```

Note: `args` contains a number of useful items, such as `troot`, `iroot`, `fwroot` (comparable
to their bash counterparts in `install.sh`, `database_host`, `client_host`, and many others)

Note: Using `localhost` in the raw configuration file is not a requirement as long as the
`replace_text` call properly injects the database host provided to the benchmark 
toolset as a command line argument.

#### A full example

Here is an example of Wicket's setup file.

```python
import subprocess
import sys
import setup_util

##################################################
# start(args, logfile, errfile)
#
# Starts the server for Wicket
# returns 0 if everything completes, 1 otherwise
##################################################
def start(args, logfile, errfile):

# setting the database url
setup_util.replace_text(args.troot + "/src/main/webapp/WEB-INF/resin-web.xml", "mysql:\/\/.*:3306", "mysql://" + args.database_host + ":3306")

# 1. Compile and package
# 2. Clean out possible old tests
# 3. Copy package to Resin's webapp directory
# 4. Start resin
try:
  subprocess.check_call("mvn clean compile war:war", shell=True, cwd="wicket", stderr=errfile, stdout=logfile)
  subprocess.check_call("rm -rf $RESIN_HOME/webapps/*", shell=True, stderr=errfile, stdout=logfile)
  subprocess.check_call("cp $TROOT/target/hellowicket-1.0-SNAPSHOT.war $RESIN_HOME/webapps/wicket.war", shell=True, stderr=errfile, stdout=logfile)
  subprocess.check_call("$RESIN_HOME/bin/resinctl start", shell=True, stderr=errfile, stdout=logfile)
  return 0
except subprocess.CalledProcessError:
  return 1

##################################################
# stop(logfile, errfile)
#
# Stops the server for Wicket
# returns 0 if everything completes, 1 otherwise
##################################################
def stop(logfile):
try:
  subprocess.check_call("$RESIN_HOME/bin/resinctl shutdown", shell=True, stderr=errfile, stdout=logfile)
  return 0
except subprocess.CalledProcessError:
  return 1
```
# Benchmark Config File

The `benchmark_config` file is used by our scripts to identify available tests - it should exist at the root of the framework directory.

Here is an example `benchmark_config` from the `Compojure` framework. There are two different tests listed for the `Compojure` framework.

    {
      "framework": "compojure",
      "tests": [{
        "default": {
          "setup_file": "setup",
          "json_url": "/compojure/json",
          "db_url": "/compojure/db/1",
          "query_url": "/compojure/db/",
          "fortune_url": "/compojure/fortune-hiccup",
          "plaintext_url": "/compojure/plaintext",
          "port": 8080,
          "approach": "Realistic",
          "classification": "Micro",
          "database": "MySQL",
          "framework": "compojure",
          "language": "Clojure",
          "orm": "Micro",
          "platform": "Servlet",
          "webserver": "Resin",
          "os": "Linux",
          "database_os": "Linux",
          "display_name": "compojure",
          "notes": "",
          "versus": "servlet"
        },
        "raw": {
          "setup_file": "setup",
          "db_url": "/compojure/dbraw/1",
          "query_url": "/compojure/dbraw/",
          "port": 8080,
          "approach": "Realistic",
          "classification": "Micro",
          "database": "MySQL",
          "framework": "compojure",
          "language": "Clojure",
          "orm": "Raw",
          "platform": "Servlet",
          "webserver": "Resin",
          "os": "Linux",
          "database_os": "Linux",
          "display_name": "compojure-raw",
          "notes": "",
          "versus": "servlet"
        }
      }]
    }

* `framework:` Specifies the framework name.
* `tests:` A list of tests that can be run for this framework. In many cases, this contains a single element for the "default" test, but additional tests can be specified.  Each test name must be unique when concatenated with the framework name. Each test will be run separately in our Rounds, so it is to your benefit to provide multiple variations in case one works better in some cases.
  * `setup_file:` The location of the [python setup file](#setup-file) that can start and stop the test, excluding the `.py` ending. If your different tests require different setup approachs, use another setup file. 
  * `json_url (optional):` The URI to the JSON test, typically `/json`
  * `db_url (optional):` The URI to the database test, typically `/db`
  * `query_url (optional):` The URI to the variable query test. The URI must be set up so that an integer can be applied to the end of the URI to specify the number of queries to run.  For example, `/query?queries=`(to yield `/query?queries=20`) or `/query/` (to yield `/query/20`)
  * `fortune_url (optional):` the URI to the fortunes test, typically `/fortune`
  * `update_url (optional):` the URI to the updates test, setup in a manner similar to `query_url` described above.
  * `plaintext_url (optional):` the URI of the plaintext test, typically `/plaintext`
  * `port:` The port the server is listening on
  * `approach (metadata):` `Realistic` or `Stripped` (see [here](http://www.techempower.com/benchmarks/#section=code&hw=peak&test=json) for a description of all metadata attributes)
  * `classification (metadata):` `Full`, `Micro`, or `Platform`
  * `database (metadata):` `MySQL`, `Postgres`, `MongoDB`, `SQLServer`, or `None`
  * `framework (metadata):` name of the framework
  * `language (metadata):` name of the language
  * `orm (metadata):` `Full`, `Micro`, or `Raw`
  * `platform (metadata):` name of the platform
  * `webserver (metadata):` name of the web-server (also referred to as the "front-end server")
  * `os (metadata):` The application server's operating system, `Linux` or `Windows`
  * `database_os (metadata):` The database server's operating system, `Linux` or `Windows`
  * `display_name (metadata):` How to render this test permutation's name on the results web site.  Some permutation names can be really long, so the display_name is provided in order to provide something more succinct.
  * `versus (optional):` The name of another test (elsewhere in this project) that is a subset of this framework.  This allows for the generation of the framework efficiency chart in the results web site. For example, Compojure is compared to "servlet" since Compojure is built on the Servlets platform.

The [requirements section](../Project-Information/Framework-Tests.md#requirements) explains the expected response for each URL as well all metadata options available. 