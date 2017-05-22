Ansible-container: look ma, no Dockerfile!
----

Creating a supportable docker container is hard. In order to succeed one should to follow [Container Best Practices](https://github.com/projectatomic/container-best-practices), write a simple yet descriptive Dockerfile, write a couple of bash scripts to make sure things are installed and probably a docker-compose files to wire several containers together and run them with correct parameters. You probably would want to use ansible to control the process and deploy the resulting image.

But wait, why not use ansible to do almost all of these parts in the playbook? This can be done using [Ansible-Container](https://github.com/ansible/ansible-container) project.


## The beast

Some time ago OSBS team has started using several containers to maintain the infrastructure. One of the most visible containers was Grafana instance, which displayed various useful information.

I simply picked up a [Grafana-XXL](https://github.com/monitoringartist/grafana-xxl) container and it worked beautifully. Until one day we needed to allow the anonymous users to view the dashboards without a login. So I decided to quickly update the Dockerfile:

```
FROM debian:jessie
MAINTAINER Jan Garaj info@monitoringartist.com

ENV \
  GRAFANA_VERSION=3.1.1-1470047149 \
  GF_PLUGIN_DIR=/grafana-plugins \
  UPGRADEALL=true

COPY ./run.sh /run.sh

RUN \
  apt-get update && \
  apt-get -y --force-yes --no-install-recommends install libfontconfig curl ca-certificates git jq && \
  curl https://grafanarel.s3.amazonaws.com/builds/grafana_${GRAFANA_VERSION}_amd64.deb > /tmp/grafana.deb && \
  dpkg -i /tmp/grafana.deb && \
  rm /tmp/grafana.deb && \
  curl -L https://github.com/tianon/gosu/releases/download/1.7/gosu-amd64 > /usr/sbin/gosu && \
  chmod +x /usr/sbin/gosu && \
  for plugin in $(curl -s https://grafana.net/api/plugins?orderBy=name | jq '.items[] | select(.internal=='false') | .slug' | tr -d '"'); do grafana-cli --pluginsDir "${GF_PLUGIN_DIR}" plugins install $plugin; done && \
  ### aion && \
  git clone https://github.com/FlukeNetworks/grafana-datasource-aion  $GF_PLUGIN_DIR/aion-datasource && \
```

Its not that complicated, but its not that easy to understand whats happening there at a glance.

First of all, its based on debian, which is not really what I'd like to support.

Then we install the grafana and gosu package - I definitely won't need gosu there.

Next, we pull in every single non-default plugin from grafana site - this is a huge overkill - I might need some, but writing a bash script to filter out the rest would be too complicated.

This is why I simply rewrote it using ansible-container.


## The beauty

First thing to do was to install ansible-container - its a simple python project:

    git clone https://github.com/ansible/ansible-container
    cd ansible-container
    git checkout develop
    sudo python setup.py install

Ansible-container is a young project, it moves fast and may break things, so I'll use ```develop``` branch.

Time to make this container a little bit more sane:

    git clone https://github.com/monitoringartist/grafana-xxl
    ansible-container init


The last command creates a new folder:

    ansible
    ├── container.yml
    ├── main.yml
    └── requirements.txt

```requirements.txt``` lists ansible modules we're going to use to build this container. I don't need anything fancy, so I'll leave the default one.

```main.yml``` will contain instructions we'll run to build a container.

The list of containers we're about to create is set in ```container.yml```. The syntax is similar to the one ```docker-compose``` uses:

    version: "1"
    services:
      grafana:
        image: centos:7
        ports:
          - "3000:3000"
        user: "grafana"
        command: ["/usr/sbin/grafana-server",
                  "--homepath=/usr/share/grafana",
                  "--config=/etc/grafana/grafana.ini",
                  "cfg:default.paths.data=/var/lib/grafana",
                  "cfg:default.paths.logs=/var/log/grafana",
                  "cfg:default.paths.plugins=/grafana-plugins"]
    registries:
      atomicregistry:
        url: atomic-registry.usersys.redhat.com:5000
        namespace: vrutkovs


We're going to create a container named ```grafana```, based on centos 7. The container would expose a 3000 port, run as grafana user and run the specified command.

Installation procedure is defined in ```main.yml``` and its fairly simple:

    - hosts: grafana
      vars:
        grafana_version: 3.1.1-1470047149
      tasks:
        - name: Upgrade all packages
          yum: name=* state=latest
        - name: Install grafana
          yum: name="https://grafanarel.s3.amazonaws.com/builds/grafana-{{ grafana_version }}.x86_64.rpm" state=present
        - name: Install plugins
          command: grafana-cli --pluginsDir /grafana-plugins plugins install {{ item }}
          with_items:
            - grafana-piechart-panel
            - alexanderzobnin-zabbix-app
            - mtanda-histogram-panel
        - name: Copy config file
          copy: src="grafana.ini" dest=/etc/grafana owner=grafana
        - name: Make sure grafana user is the owner of its dirs
          file: name={{ item }} state=directory owner=grafana recurse=true
          with_items:
            - /grafana-plugins
            - /var/lib/grafana
            - /var/log/grafana
        - name: Clean yum files
          command: yum clean all


In this file we can use standard ansible features - ```yum``` task, copy files from ```files``` folder, ```with_items``` for readability and so on.

## Build, push, run, repeat

So, how does it work? Ansible-container will pull in ```ansible-container-builder``` image, which contains ansible and all the necessary tools. A new container based on that image will start, the code will be mounted there as a volume. The builder container will start new containers for each host and run ansible instructions there. After the playbook is complete the image will be commited:

    $ sudo ansible-container build
    No DOCKER_HOST environment variable found. Assuming UNIX socket at /var/run/docker.sock
    Starting Docker Compose engine to build your images...
    Attaching to ansible_ansible-container_1
    Attaching to ansible_ansible-container_1, ansible_grafana_1
    ansible-container_1  |
    ansible-container_1  | PLAY [grafana] *****************************************************************
    ansible-container_1  |
    ansible-container_1  | TASK [setup] *******************************************************************
    ansible-container_1  | ok: [grafana]
    ansible-container_1  |
    ansible-container_1  | TASK [Upgrade all packages] ****************************************************
    ansible-container_1  | ok: [grafana]
    ansible-container_1  |
    ansible-container_1  | TASK [Install dependencies]  *********************************************************
    ansible-container_1  | ok: [grafana]
    ansible-container_1  |
    ansible-container_1  | TASK [Install plugins] *********************************************************
    ansible-container_1  | changed: [grafana] => (item=grafana-piechart-panel)
    ansible-container_1  | changed: [grafana] => (item=alexanderzobnin-zabbix-app)
    ansible-container_1  | changed: [grafana] => (item=mtanda-histogram-panel)
    ansible-container_1  |
    ansible-container_1  | TASK [Copy config file] ********************************************************
    ansible-container_1  | ok: [grafana]
    ansible-container_1  |
    ansible-container_1  | TASK [Make sure grafana user is the owner of its dirs] *************************
    ansible-container_1  | ok: [grafana] => (item=/grafana-plugins)
    ansible-container_1  | ok: [grafana] => (item=/var/lib/grafana)
    ansible-container_1  | ok: [grafana] => (item=/var/log/grafana)
    ansible-container_1  |
    ansible-container_1  | TASK [Clean yum files] *********************************************************
    ansible-container_1  | changed: [grafana]
    ansible-container_1  |  [WARNING]: Consider using yum module rather than running yum
    ansible-container_1  |
    ansible-container_1  | PLAY RECAP *********************************************************************
    ansible-container_1  | grafana                    : ok=8    changed=2    unreachable=0    failed=0   
    ansible-container_1  |
    ansible_ansible-container_1 exited with code 0
    Aborting on container exit...
    Stopping ansible_grafana_1 ... done
    Exporting built containers as images...
    Committing image...
    Exported grafana-xxl-grafana with image ID sha256:e3bfd58ae1bce0a85a96aae47e628443cbaf338f1afcccc41ae975ce8cb10d7b
    Cleaning up grafana build container...
    Cleaning up Ansible Container builder...

Docker images lists the resulting images (each image has a unique version tag):

    REPOSITORY                                                             TAG                 IMAGE ID            CREATED             SIZE
    grafana-xxl-grafana                                                    20160818105936      e3bfd58ae1bc        4 minutes ago       434.3 MB
    grafana-xxl-grafana                                                    latest              e3bfd58ae1bc        4 minutes ago       434.3 MB


We can also try it out locally:

    $ sudo ansible-container run
    No DOCKER_HOST environment variable found. Assuming UNIX socket at /var/run/docker.sock
    Attaching to ansible_ansible-container_1
    Attaching to ansible_grafana_1
    grafana_1            | t=2016-08-18T11:03:10+0000 lvl=info msg="Starting Grafana" logger=main version=3.1.1 commit=a4d2708 compiled=2016-08-01T10:20:16+0000
    grafana_1            | t=2016-08-18T11:03:10+0000 lvl=info msg="Config loaded from" logger=settings file=/usr/share/grafana/conf/defaults.ini
    grafana_1            | t=2016-08-18T11:03:10+0000 lvl=info msg="Config loaded from" logger=settings file=/etc/grafana/grafana.ini
    grafana_1            | t=2016-08-18T11:03:10+0000 lvl=info msg="Config overriden from command line" logger=settings arg="default.paths.data=/var/lib/grafana"
    grafana_1            | t=2016-08-18T11:03:10+0000 lvl=info msg="Config overriden from command line" logger=settings arg="default.paths.logs=/var/log/grafana"
    grafana_1            | t=2016-08-18T11:03:10+0000 lvl=info msg="Config overriden from command line" logger=settings arg="default.paths.plugins=/grafana-plugins"
    ...

Ansible container can also push the resulting image to a docker registry, defined in ```registries``` list of ```container.yml```:

    $ sudo ansible-container push --username vrutkovs --email vrutkovs@redhat.com --password sikrit_toahkin --push-to atomicregistry

Now we can pull in the built image on the remote docker host and run it. Unfortunately, building and running on remote docker host is not fully supported (ansible-container mounts ```ansible``` folder on ansible-builder container, so it has to exist on the remote host at the same path). If you feel this is wrong, please support me at [pull request 170](https://github.com/ansible/ansible-container/pull/170)


Ansible-container can also help with deployment to Openshift/Kubernetes:

    $ ansible-container shipit openshift  --pull-from atomic-registry.usersys.redhat.com:5000/vrutkovs

This command would create an ansible playbook, which would pull the package from the registry and create a new openshift deployment.


## Conclusion

Ansible-container is a great tool to create containers witchout messing with Dockerfiles and bash scripts. It has a few drawback currently (e.g. lack of caching), but it helps with containerizing existing Ansible deployments and gives more control over container build scripts.
