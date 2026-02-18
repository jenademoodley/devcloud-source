+++
date = 2022-06-01
draft = false
title = 'How to use ARM-based GPU EC2 instances as ECS container instances'
url = 'how-to-use-arm-based-gpu-ec2-instances-as-ecs-container-instances'
weight = 10
showAuthor = false
authors = ["jenademoodley"]
showHero = true
heroStyle = 'background'
categories = ['AWS']
tags = ['EC2', 'ECS', 'ARM']
+++

<p class="text-xs text-neutral-500 dark:text-neutral-400"> Image:
    <a class="hover:underline hover:decoration-primary-400 hover:text-primary-500" href="https://blogs.nvidia.com/blog/ngc-storefront-aws-marketplace/" title="cloud icons">NVIDIA</a>
</p>

With new EC2 instance types such as G5g, you can now use GPUs with ARM on EC2. While ECS offers some the ECS-optimized AMI as a way to quickly setup your EC2 instances for ECS workloads, you will find that the GPU optimized AMI is only for the `x86` platform, and there is no support for ARM-based GPU instances. The reason for this is quite simple; NVIDIA GPU container workloads require the `nvidia-docker` container runtime, and support is not yet available for Amazon Linux. Support is only available for CentOS 8, RHEL 8, RHEL 9, and Ubuntu versions 18.04 and above. You can review the supported distributions in [`nvidia-docker` documentation](https://nvidia.github.io/nvidia-docker/).

Given that [CentOS 8 has reached End Of Life](https://www.centos.org/centos-linux-eol/) and [RedHat has dropped support for the Docker runtime engine](https://www.techrepublic.com/article/a-matter-of-license-and-lock-in/), that leaves us only with Ubuntu. You could try and setup Docker on CentOS or RHEL, but given the lack of support this would not be advisable.

To setup an Ubuntu ARM-based GPU instance for ECS, we would need to follow the below steps.

1.    [Install NVIDIA drivers](#install-nvidia-drivers).
2.    [Install Docker and NVIDIA container runtime](#install-docker-and-nvidia-container-runtime).
3.    [Install ECS agent](#install-ecs-agent).

In my testing I used Ubuntu 20.04 on a G5g.xlarge instance, but these steps should work regardless of your distribution.

## Install NVIDIA Drivers

G5g instances currently support the Tesla driver version 470.82.01 or later. You can download the appropriate Driver directly from [Nvidia](http://www.nvidia.com/Download/Find.aspx).

You can use the below specifications to find the driver, [straight from AWS documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/install-nvidia-driver.html#nvidia-installation-options).

| Instance	| Product Type |	Product Series	| Product  |
| ---	| --- |	---	| ---  |
| G5g	| Tesla |	T-Series |	NVIDIA T4G |


In this example, I am using version “510.73.08” of the driver, the latest version of the driver at the time of writing this article. This driver can be installed with the below steps:

1. Download the driver.
    ```bash
    wget https://us.download.nvidia.com/tesla/510.73.08/NVIDIA-Linux-aarch64-510.73.08.run
    ```

2. Install gcc, make and headers.

    ```bash
    sudo apt-get update -y
    sudo apt-get install gcc make linux-headers-$(uname -r) -y
    ```

3. Run the executable.

    ```bash
    chmod +x NVIDIA-Linux-aarch64-510.73.08.run
    sudo sh ./NVIDIA-Linux-aarch64-510.73.08.run  --disable-nouveau --silent
    ```

4. (Optional) Test if GPU is detected.

    ```bash
    nvidia-smi
    ```

## Install Docker and NVIDIA container runtime

1. Download and install Docker.

    ```bash
    curl https://get.docker.com | sh   && sudo systemctl --now enable docker
    ```


2. Setup NVIDIA repository information.

    ```bash
    distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
      && curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
      && curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
            sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
            sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
    ```


3. Install NVIDIA container runtime.
    ```bash
    sudo apt-get update
    sudo apt-get install -y nvidia-docker2 nvidia-container-runtime
    sudo systemctl restart docker
    ```

4. Confirm if the NVIDIA runtime is used by Docker.
   ```bash
   sudo docker info --format '{{json .Runtimes.nvidia}}'
   ```

    You should receive the below output.
    ```bash
    {"path":"nvidia-container-runtime"}
    ```


## Install ECS agent

1.  Download the ARM-based ECS agent for Ubuntu.
    ```bash
    curl -O https://s3.us-east-1.amazonaws.com/amazon-ecs-agent-us-east-1/amazon-ecs-init-latest.arm64.deb
    ```

2. Install the ECS agent.
    ```bash
    sudo dpkg -i amazon-ecs-init-latest.arm64.deb
    ```
    
3. Configure GPU support for the agent.

    ```bash
    sudo mkdir -p /etc/ecs/
    sudo touch /etc/ecs/ecs.config
    echo "ECS_ENABLE_GPU_SUPPORT=true" | sudo tee -a /etc/ecs/ecs.config
    ```

    At this stage, you can add any additional configuration to the ecs.config file such as setting the ECS cluster.

4. Start the agent.
   ```bash
   sudo systemctl enable ecs
   sudo systemctl start ecs
   ```


And that’s it! You would see the instance in the ECS console, registered as a container instance. You can begin assigning GPUs to your containers and schedule them on these instances. If you would like to test a sample application, you can use the below ECS task definition to simply check the NVIDIA GPU with `nvidia-smi`.

```json
{
    "containerDefinitions": [
        {
        "memory": 80,
        "essential": true,
        "name": "gpu",
        "image": "nvidia/cuda:11.4.0-base-ubuntu20.04",
        "resourceRequirements": [
            {
            "type":"GPU",
            "value": "1"
            }
        ],
        "command": [
            "sh",
            "-c",
            "nvidia-smi"
        ],
        "cpu": 100
        }
    ],
    "family": "example-ecs-gpu"
}
```

## Conclusion

These steps are necessary in order to setup Ubuntu for ECS workloads on G5g instances. There may be other ARM-based GPU instances added, but this looks to be the only one for now. We can expect support for Amazon Linux once NVIDIA adds support, but there’s no confirmation when or even if that will be. It would also be nice to see additional OS support such as Rocky Linux to allow for more variety, but this ultimately depends on NVIDIA. Only time will tell what else we can use. For now, this is a working solution which you can use to setup your ECS workloads on ARM-based GPU EC2 instances.