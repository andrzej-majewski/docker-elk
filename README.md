# Docker EFK stack


Run the latest version(6.2.1) of the EFK (Elasticsearch, Fluentd, Kibana) stack with Docker Swarm. And sample Java standalone application.

It will give you the ability to analyze any data set by using the searching/aggregation capabilities of Elasticsearch
and the visualization power of Kibana.

Based on the official Docker images:

* [elasticsearch](https://github.com/elastic/elasticsearch-docker)
* [fluentd](https://github.com/fluent/fluentd-docker-image)
* [kibana](https://github.com/elastic/kibana-docker)

## Contents

1. [Requirements](#requirements)
   * [Host setup](#host-setup)
   * [SELinux](#selinux)
2. [Getting started](#getting-started)
   * [Bringing up the stack](#bringing-up-the-stack)
   * [Initial setup](#initial-setup)
3. [Configuration](#configuration)
   * [How can I tune the Kibana configuration?](#how-can-i-tune-the-kibana-configuration)
   * [How can I tune the Logstash configuration?](#how-can-i-tune-the-logstash-configuration)
   * [How can I tune the Elasticsearch configuration?](#how-can-i-tune-the-elasticsearch-configuration)
   * [How can I scale out the Elasticsearch cluster?](#how-can-i-scale-up-the-elasticsearch-cluster)
4. [Storage](#storage)
   * [How can I persist Elasticsearch data?](#how-can-i-persist-elasticsearch-data)
5. [Extensibility](#extensibility)
   * [How can I add plugins?](#how-can-i-add-plugins)
   * [How can I enable the provided extensions?](#how-can-i-enable-the-provided-extensions)
6. [JVM tuning](#jvm-tuning)
   * [How can I specify the amount of memory used by a service?](#how-can-i-specify-the-amount-of-memory-used-by-a-service)
   * [How can I enable a remote JMX connection to a service?](#how-can-i-enable-a-remote-jmx-connection-to-a-service)

## Requirements

### Host setup

1. Install [Docker](https://www.docker.com/community-edition#/download) version **1.10.0+**
2. Install [Docker Compose](https://docs.docker.com/compose/install/) version **1.6.0+**
3. Install [Docker Swarm](https://docs.docker.com/engine/swarm/swarm-tutorial/)
3. Clone this repository

### SELinux

On distributions which have SELinux enabled out-of-the-box you will need to either re-context the files or set SELinux
into Permissive mode in order for docker-elk to start properly. For example on Redhat and CentOS, the following will
apply the proper context:

```console
$ chcon -R system_u:object_r:admin_home_t:s0 docker-elk/
```

## Usage

### Build Fluentd docker image


```console
$ build -t  dockerelk_fluentd:latest .


amajewski$ docker build -t  dockerelk_fluentd:latest .
Sending build context to Docker daemon  11.26kB
Step 1/4 : FROM fluent/fluentd:v0.14.25
 ---> ee0ba3eb7d98
Step 2/4 : RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-rdoc", "--no-ri", "--version", "2.6.1"]
 ---> Using cache
 ---> 35f514b324ef
Step 3/4 : RUN ["gem", "install", "fluent-plugin-detect-exceptions", "--no-rdoc", "--no-ri", "--version", "0.0.8"]
 ---> Using cache
 ---> 573fb9673e1b
Step 4/4 : RUN ["gem", "install", "fluent-plugin-concat", "--no-rdoc", "--no-ri", "--version", "2.2.0"]
 ---> Using cache
 ---> 97ce77a72f80
Successfully built 97ce77a72f80
Successfully tagged dockerelk_fluentd:latest

```

### Bringing up the stack

Start Docker swarm using `docker-swarm`:

```console
$ docker swarm init


Swarm initialized: current node (vip5mzphn17yps0hhuxb0wfe2) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-42f05bnmx64vxciqwhc5rp055ot5elgy73r3ec9zacsm2zbv8b-ek38a6uipjxulry67rx0towy6 192.168.65.3:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

```


Start the EFK stack using `docker-stack`:

```console
$ docker stack deploy  --compose-file docker-stack.yml elk

Creating network elk_elk
Creating service elk_elasticsearch
Creating service elk_kibana
Creating service elk_fluentd
Creating service elk_samplejms

docker ps

CONTAINER ID        IMAGE                                                     COMMAND                  CREATED             STATUS              PORTS                 NAMES
bc2f1cb67e67        dockerelk_fluentd:latest                                  "/bin/entrypoint.sh …"   46 seconds ago      Up 47 seconds       5140/tcp, 24224/tcp   elk_fluentd.1.tpr109svkfxlcigx9p2td16e1
cfb57551bb3a        docker.elastic.co/kibana/kibana-oss:6.2.1                 "/bin/bash /usr/loca…"   51 seconds ago      Up 52 seconds       5601/tcp              elk_kibana.1.qugeg8yap6yin6qpgnx5nzb15
1c9612b45f5e        docker.elastic.co/elasticsearch/elasticsearch-oss:6.2.1   "/usr/local/bin/dock…"   55 seconds ago      Up 56 seconds       9200/tcp, 9300/tcp    elk_elasticsearch.1.o2opsdrxi9747lei3bkkwpsdc

docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                                                     PORTS
iwlyf81bmms7        elk_elasticsearch   replicated          1/1                 docker.elastic.co/elasticsearch/elasticsearch-oss:6.2.1   *:9200->9200/tcp,*:9300->9300/tcp
ks2lcks6d2nz        elk_fluentd         replicated          1/1                 dockerelk_fluentd:latest                                  *:24224->24224/tcp,*:24224->24224/udp
0g4ek1l0s7nf        elk_kibana          replicated          1/1                 docker.elastic.co/kibana/kibana-oss:6.2.1                 *:5601->5601/tcp
m9l8rokp1ojt        elk_samplejms       replicated          0/1                 ikasan-boot-jms:2.0.0-rc3                                 *:8080->8080/tcp

```


Give Kibana a few seconds to initialize, then access the Kibana web UI by hitting
[http://localhost:5601](http://localhost:5601) with a web browser.

By default, the stack exposes the following ports:
* 24224: Fluentd TCP input.
* 9200: Elasticsearch HTTP
* 9300: Elasticsearch TCP transport
* 5601: Kibana

**WARNING**: If you're using `boot2docker`, you must access it via the `boot2docker` IP address instead of `localhost`.

**WARNING**: If you're using *Docker Toolbox*, you must access it via the `docker-machine` IP address instead of
`localhost`.

Now that the stack is running, you will want to inject some log entries. The shipped Logstash configuration allows you
to send content via TCP:

```console
$ nc localhost 24224 < /path/to/logfile.log
```

#### Check JVM application logs on fluentd

Check if fluentd container and java container is running

```console
$  docker ps
CONTAINER ID        IMAGE                                                     COMMAND                  CREATED             STATUS              PORTS                 NAMES
0c8464b6b82f        ikasan-boot-jms:2.0.0-rc3                                 "java -Djava.securit…"   2 minutes ago       Up 2 minutes                              elk_samplejms.1.64fubvvg29arqhkb60j2utio2
bc2f1cb67e67        dockerelk_fluentd:latest                                  "/bin/entrypoint.sh …"   5 minutes ago       Up 5 minutes        5140/tcp, 24224/tcp   elk_fluentd.1.tpr109svkfxlcigx9p2td16e1

```

Check fluentd picking up logs

```console
$  docker logs -f `docker ps -a| grep fluentd | head -1 | cut -d " " -f1`

2018-02-22 11:07:50 +0000 [info]: parsing config file is succeeded path="/fluentd/etc/fluent.conf"
2018-02-22 11:07:50 +0000 [info]: 'flush_interval' is configured at out side of <buffer>. 'flush_mode' is set to 'interval' to keep existing behaviour
2018-02-22 11:07:50 +0000 [info]: using configuration file: <ROOT>
  <source>
    @type forward
    port 24224
    bind "0.0.0.0"
  </source>
  <filter **>
    @type concat
    key "log"
    stream_identity_key "container_id"
    multiline_start_regexp "/^\\d{4}-\\d{1,2}-\\d{1,2} \\d{2}:\\d{2}:\\d{2}.\\d{1,3}/"
    use_first_timestamp true
    flush_interval 3s
    timeout_label "@processdata"
  </filter>
  <match **>
    @type relabel
    @label @processdata
  </match>
  <label @processdata>
    <match **>
      @type copy
      <store>
        @type "elasticsearch"
        host "elasticsearch"
        port 9200
        logstash_format true
        logstash_prefix "fluentd"
        logstash_dateformat "%Y%m%d"
        include_tag_key true
        type_name "application_log"
        tag_key "@log_name"
        flush_interval 1s
        <buffer>
          flush_interval 1s
        </buffer>
      </store>
      <store>
        @type "stdout"
      </store>
    </match>
  </label>
</ROOT>
2018-02-22 11:07:50 +0000 [info]: starting fluentd-0.14.25 pid=5 ruby="2.3.5"
2018-02-22 11:07:50 +0000 [info]: spawn command to main:  cmdline=["/usr/bin/ruby", "-Eascii-8bit:ascii-8bit", "/usr/bin/fluentd", "-c", "/fluentd/etc/fluent.conf", "-p", "/fluentd/plugins", "--under-supervisor"]
2018-02-22 11:07:50 +0000 [info]: gem 'fluent-plugin-concat' version '2.2.0'
2018-02-22 11:07:50 +0000 [info]: gem 'fluent-plugin-detect-exceptions' version '0.0.8'
2018-02-22 11:07:50 +0000 [info]: gem 'fluent-plugin-elasticsearch' version '2.6.1'
2018-02-22 11:07:50 +0000 [info]: gem 'fluentd' version '0.14.25'
2018-02-22 11:07:50 +0000 [info]: adding match in @processdata pattern="**" type="copy"
2018-02-22 11:07:51 +0000 [info]: #0 'flush_interval' is configured at out side of <buffer>. 'flush_mode' is set to 'interval' to keep existing behaviour
2018-02-22 11:07:51 +0000 [info]: adding filter pattern="**" type="concat"
2018-02-22 11:07:51 +0000 [info]: adding match pattern="**" type="relabel"
2018-02-22 11:07:51 +0000 [info]: adding source type="forward"
2018-02-22 11:07:51 +0000 [info]: #0 starting fluentd worker pid=15 ppid=5 worker=0
2018-02-22 11:07:51 +0000 [info]: #0 listening port port=24224 bind="0.0.0.0"
2018-02-22 11:07:51 +0000 [info]: #0 fluentd worker is now running worker=0
2018-02-22 11:07:51.088394993 +0000 fluent.info: {"worker":0,"message":"fluentd worker is now running worker=0"}
2018-02-22 11:07:53 +0000 [info]: #0 Connection opened to Elasticsearch cluster => {:host=>"elasticsearch", :port=>9200, :scheme=>"http"}
2018-02-22 11:07:53.104054759 +0000 fluent.info: {"message":"Connection opened to Elasticsearch cluster => {:host=>\"elasticsearch\", :port=>9200, :scheme=>\"http\"}"}
2018-02-22 11:10:49.000000000 +0000 samplejms: {"source":"stderr","log":"SLF4J: Class path contains multiple SLF4J bindings.","container_id":"0c8464b6b82f651c81224399bceb5255ae5177439025ab3ac3a88eeec891fa82","container_name":"/elk_samplejms.1.64fubvvg29arqhkb60j2utio2"}
2018-02-22 11:10:49.000000000 +0000 samplejms: {"log":"SLF4J: Found binding in [jar:file:/app.jar!/BOOT-INF/lib/logback-classic-1.1.11.jar!/org/slf4j/impl/StaticLoggerBinder.class]","container_id":"0c8464b6b82f651c81224399bceb5255ae5177439025ab3ac3a88eeec891fa82","container_name":"/elk_samplejms.1.64fubvvg29arqhkb60j2utio2","source":"stderr"}
2018-02-22 11:10:49.000000000 +0000 samplejms: {"container_id":"0c8464b6b82f651c81224399bceb5255ae5177439025ab3ac3a88eeec891fa82","container_name":"/elk_samplejms.1.64fubvvg29arqhkb60j2utio2","source":"stderr","log":"SLF4J: Found binding in [jar:file:/app.jar!/BOOT-INF/lib/activemq-all-5.13.4.jar!/org/slf4j/impl/StaticLoggerBinder.class]"}
2018-02-22 11:10:49.000000000 +0000 samplejms: {"source":"stderr","log":"SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.","container_id":"0c8464b6b82f651c81224399bceb5255ae5177439025ab3ac3a88eeec891fa82","container_name":"/elk_samplejms.1.64fubvvg29arqhkb60j2utio2"}
2018-02-22 11:10:49.000000000 +0000 samplejms: {"log":"SLF4J: Actual binding is of type [ch.qos.logback.classic.util.ContextSelectorStaticBinder]","container_id":"0c8464b6b82f651c81224399bceb5255ae5177439025ab3ac3a88eeec891fa82","container_name":"/elk_samplejms.1.64fubvvg29arqhkb60j2utio2","source":"stderr"}


```



## Initial setup

### Default Kibana index pattern creation

When Kibana launches for the first time, it is not configured with any index pattern.

#### Via the Kibana web UI

**NOTE**: You need to inject data into Logstash before being able to configure a Logstash index pattern via the Kibana web
UI. Then all you have to do is hit the *Create* button.

Refer to [Connect Kibana with
Elasticsearch](https://www.elastic.co/guide/en/kibana/current/connect-to-elasticsearch.html) for detailed instructions
about the index pattern configuration.

#### On the command line

Create an index pattern via the Kibana API:

```console
$ curl -XPOST -D- 'http://localhost:5601/api/saved_objects/index-pattern' \
    -H 'Content-Type: application/json' \
    -H 'kbn-version: 6.2.1' \
    -d '{"attributes":{"title":"logstash-*","timeFieldName":"@timestamp"}}'
```

The created pattern will automatically be marked as the default index pattern as soon as the Kibana UI is opened for the first time.

## Configuration

**NOTE**: Configuration is not dynamically reloaded, you will need to restart the stack after any change in the
configuration of a component.

### How can I tune the Kibana configuration?

The Kibana default configuration is stored in `kibana/config/kibana.yml`.

It is also possible to map the entire `config` directory instead of a single file.


### How can I tune the Elasticsearch configuration?

The Elasticsearch configuration is stored in `elasticsearch/config/elasticsearch.yml`.

You can also specify the options you want to override directly via environment variables:

```yml
elasticsearch:

  environment:
    network.host: "_non_loopback_"
    cluster.name: "my-cluster"
```

### How can I scale out the Elasticsearch cluster?

Follow the instructions from the Wiki: [Scaling out
Elasticsearch](https://github.com/deviantony/docker-elk/wiki/Elasticsearch-cluster)

## Storage

### How can I persist Elasticsearch data?

The data stored in Elasticsearch will be persisted after container reboot but not after container removal.

In order to persist Elasticsearch data even after removing the Elasticsearch container, you'll have to mount a volume on
your Docker host. Update the `elasticsearch` service declaration to:

```yml
elasticsearch:

  volumes:
    - /path/to/storage:/usr/share/elasticsearch/data
```

This will store Elasticsearch data inside `/path/to/storage`.

**NOTE:** beware of these OS-specific considerations:
* **Linux:** the [unprivileged `elasticsearch` user][esuser] is used within the Elasticsearch image, therefore the
  mounted data directory must be owned by the uid `1000`.
* **macOS:** the default Docker for Mac configuration allows mounting files from `/Users/`, `/Volumes/`, `/private/`,
  and `/tmp` exclusively. Follow the instructions from the [documentation][macmounts] to add more locations.

[esuser]: https://github.com/elastic/elasticsearch-docker/blob/016bcc9db1dd97ecd0ff60c1290e7fa9142f8ddd/templates/Dockerfile.j2#L22
[macmounts]: https://docs.docker.com/docker-for-mac/osxfs/

## Extensibility

### How can I add plugins?

To add plugins to any ELK component you have to:

1. Add a `RUN` statement to the corresponding `Dockerfile` (eg. `RUN logstash-plugin install logstash-filter-json`)
2. Add the associated plugin code configuration to the service configuration (eg. Logstash input/output)
3. Rebuild the images using the `docker-compose build` command

### How can I enable the provided extensions?

A few extensions are available inside the [`extensions`](extensions) directory. These extensions provide features which
are not part of the standard Elastic stack, but can be used to enrich it with extra integrations.

The documentation for these extensions is provided inside each individual subdirectory, on a per-extension basis. Some
of them require manual changes to the default ELK configuration.

## JVM tuning

### How can I specify the amount of memory used by a service?

By default, both Elasticsearch and Logstash start with [1/4 of the total host
memory](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#default_heap_size) allocated to
the JVM Heap Size.

The startup scripts for Elasticsearch and Logstash can append extra JVM options from the value of an environment
variable, allowing the user to adjust the amount of memory that can be used by each component:

| Service       | Environment variable |
|---------------|----------------------|
| Elasticsearch | ES_JAVA_OPTS         |
| Logstash      | LS_JAVA_OPTS         |

To accomodate environments where memory is scarce (Docker for Mac has only 2 GB available by default), the Heap Size
allocation is capped by default to 256MB per service in the `docker-compose.yml` file. If you want to override the
default JVM configuration, edit the matching environment variable(s) in the `docker-compose.yml` file.

For example, to increase the maximum JVM Heap Size for Logstash:

```yml
logstash:

  environment:
    LS_JAVA_OPTS: "-Xmx1g -Xms1g"
```

### How can I enable a remote JMX connection to a service?

As for the Java Heap memory (see above), you can specify JVM options to enable JMX and map the JMX port on the docker
host.

Update the `{ES,LS}_JAVA_OPTS` environment variable with the following content (I've mapped the JMX service on the port
18080, you can change that). Do not forget to update the `-Djava.rmi.server.hostname` option with the IP address of your
Docker host (replace **DOCKER_HOST_IP**):

```yml
logstash:

  environment:
    LS_JAVA_OPTS: "-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.port=18080 -Dcom.sun.management.jmxremote.rmi.port=18080 -Djava.rmi.server.hostname=DOCKER_HOST_IP -Dcom.sun.management.jmxremote.local.only=false"
```
