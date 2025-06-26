+++
date = 2020-07-24
draft = false
title = 'ECS - Dynamic Port Mapping'
url = 'ecs-dynamic-port-mapping'
description = 'How to use ECS Dynamic Port Mapping'
weight = 10
showAuthor = false
authors = ["jenademoodley"]
showHero = true
heroStyle = 'big'
summary = 'Summary'
categories = ['AWS']
tags = ['ECS', 'ELB', 'ALB']
+++

## The Port Mapping Problem

A popular function of containers is to run a service such as a web or application server to which clients connect to using HTTP, TCP or socket connections. By default, Docker will allocate a port from the host to the container from the ephemeral port range, but you can manually map or expose a specific host port, such as mapping port 80 on the host to port 8080 on the container. This works fine for individual containers and tasks, but if you need to run another container of the same type, you would then need another host as you cannot bind more than one container to a specific host port (i.e. if there is a container or service listening on a specific port such as port 80, no other service or container can use or bind that specific port.)

Additionally, if your container is lightweight and uses minimal CPU and memory resources, this results in resource wastage as you would need to launch an entirely new instance simply to cater for this port conflict. If you wish to configure reactive scaling (i.e. scale the number of containers due to demand), this is also a blocking factor as this means that you either need additional EC2 instances on standby (increased cost), or spin up a new instance in accordance to demand (slower scale time). You would have to accurately map and plan your port usage on your EC2 hosts such that you can launch your containers without port conflicts, which increases management overhead.

## Enter Dynamic Port Mapping

If you are going to be using multiple containers to host a service, you would also need to configure a suitable and robust solution in order to access each of these containers. When considering HTTP requests, a popular choice in AWS is to use the Application Load Balancer (ALB), often called a Layer 7 load balancer (referring to layer 7 Application layer in the OSI model, hence the name). An ALB will allow you to effectively distribute (or balance) requests to multiple targets to which it is connected to. While the port on the ALB to which you will issue requests to will always remain the same (port 80 for HTTP requests, and port 443 for HTTPS requests), the back end targets can each be listening on a different port, for example one target can be listening on port 8080, another can be listening on port 8000, etc. Multiple targets can even belong on the same EC2 instance, provided that the ports are different.

Using this functionality, we can apply this behaviour to containers. We can run the containers without specifying the port mapping (i.e. allow Docker to map the container port to an ephemeral port on the host), and then configure these containers to be targets for the ALB. This would allow you to run multiple containers of the same type, on the same EC2 instance, effectively reducing cost and management overhead. The ECS service is also capable of determining which host port your container has been bound to, and can configure the target registration for that specific port and container on the ALB, which allows for a seamless configuration.

## Setting up Dynamic Port Mapping

Dynamic Port Mapping is relatively easy to configure. From the ECS service side, all you need to do is set the "Host" port in the container mapping to "0". A JSON snippet of the task definition would therefore look like this:

```json
"portMappings": [
        {
          "hostPort": 0,
          "protocol": "tcp",
          "containerPort": 80
        }
      ]
```

However, additional configurations may need to be done on your VPC and container instances in order to allow the connections to successfully reach the container. As the containers will be mapped to the ephemeral ports of the host, you would need to ensure the following criteria is met:

1. The security groups attached to your container instances or tasks allow for incoming connections on the ephemeral port range from the ALB nodes.
2. The NACLs used by the subnet in which your tasks or container instances are located must allow inbound and outbound connections on the ephemeral port range.

The ephemeral port range used by Docker versions 1.6.0 and later will refer to the `/proc/sys/net/ipv4/ip_local_port_range` kernel parameter on the host. If this kernel parameter is not available, or if you are using an earlier version of Docker, the default ephemeral port range is used, which ranges from ports 49153 through 65535. In general, ports below 32768 fall outside of the ephemeral port range. As a general rule of thumb, you can allow incoming connections from ports 32768 through 65535 in order to account for both scenarios and avoid any issues.

## Considerations

As you can now have multiple containers running on a single host to which you can connect to using your ALB, a logical consideration is having too many containers on the same host. This can cause resource contention, and can cause containers to be stopped if the configurations are too restrictive. You should actively configure task and container memory and CPU limits so that each container will be given ample resources on the host in order to function effectively.