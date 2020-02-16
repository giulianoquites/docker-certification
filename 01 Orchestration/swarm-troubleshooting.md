# Orchestration > Identify​ ​the​ ​steps​ ​needed​ ​to​ ​troubleshoot​ ​a​ ​service​ ​not​ ​deploying

[Back](./ReadMe.md)

Troubleshooting methodology, like any other technology, is determined by the class of problem or failure mode of an issue. Issues are usually discovered (by application owners or users) at the top of the stack so errors can be traced down from there. Regardless if issues are found at the top or the bottom of the stack, tracing through to the other end will reveal the underlying cause or resulting effects. Examples of issues that can start somewhere in the stack and propogate upwards to impact the applications includes things like the following:

* Physical node failures
* Local storage volume filling to capacity
* OS kernel panics
* Exhaustion of file descriptors

![full stack](./full-stack.png)

## Viewing Software as an Onion
Inspecting software components is very much like peeling an onion, start from the outside and work your way in. Encapsulated software objects wrap each other and inspecting these object, progressively working inwards, can help build a picture of the current state of applications.

In Docker, there are many layers of encapsulation: the OS kernel, containers, tasks that encapsulate containers as units of work, services or Pods that represent application components, and stacks that represent full applications. All of these are first class objects in Kubernetes and Swarm that can be inspected independently.

![Onion](./onion.png)

## Viewing Software as a Flow
Many issues in distributed computing can be traced through multiple disparate components. These could be different containers or hosts, typically connected via an application. These distributed processes can be viewed as flows. When troubleshooting, the full end-to-end path should be considered, starting at either end and working towards the other.

Examples of issues that can be viewed as a flow are:

* Network partitions
* DNS resolution
* Distributed multi-container application issues
* Loss of quorum for consensus-based systems

![Flow](./flow.png)

## Docker Troubleshooting Best Practices

### Using a Client Bundle
The UCP client bundle sets the context for Docker engine and kubectl. When interacting with a UCP cluster, you should use the client bundle.

It is a best practice to use the UCP client bundle whenever troubleshooting UCP clusters.

* Client bundles apply your account permissions so that you are prevented from causing harm to infrastructure containers and other important pieces of the stack.
* Client bundles allow you to run commands that apply to the entire cluster, without having to log into specific nodes.

The following shows how to download a client bundle and apply it to your local Docker client:

```
UCP_URL=your_ucp_url
USER=your_ucp_username
PASSWORD=*****your_ucp_password

AUTHTOKEN=$(curl -sk -d '{"username":"$USER","password":"$PASSWORD"}' https://UCP_URL/auth/login | jq -r .auth_token)

# Download the client certificate bundle
curl -k -H "Authorization: Bearer $AUTHTOKEN" https://$UCP_URL/api/clientbundle -o bundle.zip
```

### Troubleshooting from the Service Inward
Using the concept of "software as an onion," it's a very common flow to troubleshoot Swarm first class objects from the outside in. Issues that are found via misbehaving applications are commonly troubleshooted in this way.

This is the general flow from the outside working in:

```
$ docker service ls
$ docker service ps <service>
$ docker service inspect <service>
$ docker inspect <task>
$ docker inspect <container>
$ docker logs <container>
```

In the following examples we'll use the ucp-agent as our problem service

### docker service ls
docker service ls tells us the full list of service running in Swarm. It's very helpful for showing whether desired amount of replicas for a service are actually running. For example:

```
docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                          PORTS
o1rttg33awmd        ucp-agent           global              2/2                 docker/ucp-agent:2.2.4
```

### What to Look For
2/2 tells us that of the 2 replicas that are desired, there are two currently running.

### docker service inspect
docker service inspect tells us all the specified configuration values for a given service. For example:

```
$ docker service inspect ucp-agent
[
    {
        "ID": "x8vlfxey5zxqt69mlydbil6yt",
        "Version": {
            "Index": 176
        },
        "CreatedAt": "2017-11-10T23:51:39.6345829Z",
        "UpdatedAt": "2017-11-13T00:33:46.611747454Z",
        "Spec": {
            "Name": "ucp-agent",
            "Labels": {
                "com.docker.ucp.InstanceID": "in9h5oi63hmwqof3dfm5n47z5",
                "com.docker.ucp.version": "2.2.4"
            },
            "TaskTemplate": {
                "ContainerSpec": {
                    "Image": "docker/ucp-agent:2.2.4@sha256:bbee2b38e355e613c6b740698fba887d65f77b3c69e81bd9eebd4ae6665e43c5",
                    "Labels": {
                        "com.docker.ucp.InstanceID": "in9h5oi63hmwqof3dfm5n47z5",
                        "com.docker.ucp.version": "2.2.4"
                    },
                    "Command": [
                        "/bin/ucp-agent",
                        "agent"
                    ],
                    "Env": [
                        "IMAGE_VERSION=2.2.4",

 ...
```

### What to Look For
* CreatedAt signifies when the service was created. This answers the question "Was the service deployed right around when the issue started or was the service already in steady-state?"
* UpdatedAt similarly signifies if there were any changes to the service that can correlate with the start of an issue somewhere else.
* Labels have a large variety of uses from acting as scheduling constraints to simple metadata. These are important to look at.

These values can be retrieved individually with the following commands:

```
docker service inspect ucp-agent | jq -r '.[].CreatedAt'
2017-11-10T23:51:39.6345829Z

docker service inspect ucp-agent | jq -r '.[].UpdatedAt'
2017-11-13T00:33:46.611747454Z
```
JSON representation is useful with tools like jq that pull out specific values from JSON data.

### docker service ps
This command displays a list of tasks for a given service. This is helpful in determining if there were failures in the past or if tasks are currently running or restarting. Swarm stores a certain amount of history for a service which shows some tasks whose containers are no longer running. For example:

```
$ docker service ps ucp-agent
ID                  NAME                                      IMAGE                    NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
pw6u97k0th7q        ucp-agent.ung2hozu917ekkrx8zy3kxt2f       docker/ucp-agent:2.2.4   ip-172-31-17-19     Running             Running 3 minutes ago
tzi0i0nuwnc1         \_ ucp-agent.ung2hozu917ekkrx8zy3kxt2f   docker/ucp-agent:2.2.4   ip-172-31-17-19     Shutdown            Complete 3 minutes ago
2k9mk8smec4u        ucp-agent.2z1mvt13mcil48a2qq9hwou2g       docker/ucp-agent:2.2.4   ip-172-31-31-50     Running             Running 4 minutes ago
a86qcrb6ekyu         \_ ucp-agent.2z1mvt13mcil48a2qq9hwou2g   docker/ucp-agent:2.2.4   ip-172-31-31-50     Shutdown            Complete 4 minutes ago
```

### What to Look For
A large list of failed tasks and tasks that are being created and exiting frequently are indicative of a restart loop. The container is coming up, exiting, and being rescheduled again in quick succession.

All Docker commands can use the --format "{{json .}}" option to print the output in JSON representation. For example:
```
$ docker service ps ucp-agent --format "{{json .}}"
{"CurrentState":"Running 9 minutes ago","DesiredState":"Running","Error":"","ID":"pw6u97k0th7q","Image":"docker/ucp-agent:2.2.4","Name":"ucp-agent.ung2hozu917ekkrx8zy3kxt2f","Node":"ip-172-31-17-19","Ports":""}
{"CurrentState":"Complete 9 minutes ago","DesiredState":"Shutdown","Error":"","ID":"tzi0i0nuwnc1","Image":"docker/ucp-agent:2.2.4","Name":"ucp-agent.ung2hozu917ekkrx8zy3kxt2f","Node":"ip-172-31-17-19","Ports":""}
{"CurrentState":"Running 10 minutes ago","DesiredState":"Running","Error":"","ID":"2k9mk8smec4u","Image":"docker/ucp-agent:2.2.4","Name":"ucp-agent.2z1mvt13mcil48a2qq9hwou2g","Node":"ip-172-31-31-50","Ports":""}
{"CurrentState":"Complete 10 minutes ago","DesiredState":"Shutdown","Error":"","ID":"a86qcrb6ekyu","Image":"docker/ucp-agent:2.2.4","Name":"ucp-agent.2z1mvt13mcil48a2qq9hwou2g","Node":"ip-172-31-31-50","Ports":""}
```

We use JSON output and also filter for tasks that should be running to produce the following list of Task IDs. This can be helpful if we want to loop through large lists of tasks or containers to pull out specific data.

```
docker service ps ucp-agent --format "{{json .}}" --filter "desired-state=running" | jq -r .ID
pw6u97k0th7q
2k9mk8smec4u
```

### docker inspect
Using the Task IDs from docker service ps, we can pull out the container IDs to identify which containers belong to a given service. Once we have the container ID for a given task, we can get container logs or view specific information about the container configuration.

```
$ docker inspect pw6u97k0th7q | jq -r '.[].Status.ContainerStatus.ContainerID'
90cbc98062c82e7fddac3e883f9dac26773bf426704dfc3307abc5b327eb304d
```

### docker inspect <container>
The output from docker inspect <container> is very large and detailed, but it gives us every container configuration parameter, which allows us to dive very deep into specific issues. The following sections break up the output to make it easier to understand.

### Container State
```
docker inspect 90cbc98062c8 | jq '.[].State'
{
  "Status": "running",
  "Running": true,
  "Paused": false,
  "Restarting": false,
  "OOMKilled": false,
  "Dead": false,
  "Pid": 2809,
  "ExitCode": 0,
  "Error": "",
  "StartedAt": "2017-11-18T04:52:07.101718306Z",
  "FinishedAt": "0001-01-01T00:00:00Z"
}
```

* StartedAt displays the time when the container was scheduled.
* FinishedAt displays when the container was exited or was killed. This is useful for understanding the timeline of issues.
* ExitCode and Error provide information about why a container has exited.
* OOMKilled may be used to indicate a memory pressure issue.

### Node
```
docker inspect 90cbc98062c8 | jq '.[].Node'
{
  "ID": "I2B6:36YM:QBX5:7T4V:VGJU:UNFF:MW6V:GOYG:7WAG:MWK6:SJNL:3ZGR|172.31.17.19:12376",
  "IP": "172.31.17.19",
  "Addr": "172.31.17.19:12376",
  "Name": "ip-172-31-17-19",
  "Cpus": 1,
  "Memory": 2095976448,
  "Labels": {
    "kernelversion": "4.4.0-1022-aws",
    "operatingsystem": "Ubuntu 16.04.2 LTS",
    "ostype": "linux",
    "storagedriver": "aufs"
  }
}
```

* IP displays the IP address of any host interfaces.
* Cpus and Memory indicate the host resources.

### Network Settings
```
$ docker inspect 90cbc98062c8 | jq '.[].NetworkSettings'
{
  "Bridge": "",
  "SandboxID": "c352709cf41243e3293c9a8dac563ae10d1a84dea085dd36e259324e64f2a47f",
  "HairpinMode": false,
  "LinkLocalIPv6Address": "",
  "LinkLocalIPv6PrefixLen": 0,
  "Ports": {
    "2376/tcp": null
  },
  "SandboxKey": "/var/run/docker/netns/c352709cf412",
  "SecondaryIPAddresses": null,
  "SecondaryIPv6Addresses": null,
  "EndpointID": "e75090016d7776845598dad468afb0e34a772668e5d976f216d15d187596aecb",
  "Gateway": "172.17.0.1",
  "GlobalIPv6Address": "",
  "GlobalIPv6PrefixLen": 0,
  "IPAddress": "172.17.0.2",
  "IPPrefixLen": 16,
  "IPv6Gateway": "",
  "MacAddress": "02:42:ac:11:00:02",
  "Networks": {
    "bridge": {
      "IPAMConfig": null,
      "Links": null,
      "Aliases": null,
      "NetworkID": "b7df85b0dd79859b9e746e874439ab04bffaaf902899420b06e6e6f557b1977b",
      "EndpointID": "e75090016d7776845598dad468afb0e34a772668e5d976f216d15d187596aecb",
      "Gateway": "172.17.0.1",
      "IPAddress": "172.17.0.2",
      "IPPrefixLen": 16,
      "IPv6Gateway": "",
      "GlobalIPv6Address": "",
      "GlobalIPv6PrefixLen": 0,
      "MacAddress": "02:42:ac:11:00:02",
      "DriverOpts": null
    }
  }
}
```

* SandboxKey indicates the location and name of the network namespace used by this container.
* IPAddress indicates the IP address of the primary interface of the container.
* Ports lists all of the ports that are exposed for this container.
* Networks lists all of the networks that this container is attached to.





