+++
date = 2023-12-07
draft = false
title = 'How to configure cgroupv2 on Amazon Linux 2'
url = 'how-to-configure-cgroupv2-on-amazon-linux-2'
weight = 10
showAuthor = false
authors = ["jenademoodley"]
showHero = true
heroStyle = "big"
featureimagecaption = "[Image: SoByt](https://www.sobyte.net/post/2022-07/blkio-cgroup/)"
categories = ['AWS']
tags = ['EC2', 'AMAZON LINUX']
+++


{{< alert >}}
**Note!** Amazon Linux 2 will reach end of life on June 30, 2026. For more information, see [Amazon Linux 2 FAQs](https://aws.amazon.com/amazon-linux-2/faqs/#topic-0).

It is recommended to use Amazon Linux 2023 which has cgroupv2 installed by default.
{{< /alert >}}

Amazon Linux 2 is the current default or vanilla Amazon Linux version currently used for AWS workloads. However, when using this version, one would notice that this OS still uses cgroup version 1. Given that cgroupv2 has been around since October 2015 and that certain functions such as [MemoryQoS in Kubernetes](https://kubernetes.io/blog/2021/11/26/qos-memory-resources/) relies on cgroupv2. As such, many users would like to enable cgroupv2 on Amazon Linux 2.


## Why is cgroupv2 not enabled on Amazon Linux 2?

The reason for the discrepancy is due to the version of systemd included in Amazon Linux 2. At present, the latest version of systemd available on Amazon Linux 2 is `219`, however the recommended systemd version  is `244` or later. Older systemd does not support delegation of `cpuset` controller. This is why Amazon Linux 2 still uses cgroup version 1.

However, this also shows us the solution to enable cgroupv2; by updating the version of systemd on Amazon Linux 2.

## How can I update systemd on Amazon Linux 2

At present, version `219` is the latest version of systemd as mentioned above, and AWS has not provided any indication of when a more up to date version would be released, if at all. As such, we can install an updated version of systemd by installing from source. The steps to install an updated systemd from source (tested on a current version of the Amazon Linux 2 AMI) are as below.

1. Switch to root user.
    ```bash
    sudo -i
    ```

2. Install kernel version 5.10 or later.
   ```bash
   amazon-linux-extras install kernel-5.10 -y
   ```

   Take note of the exact kernel version installed as we will use this in a later command. At the time of writing, I installed kernel version `5.10.201-191.748.amzn2.x86_64`.

    ```
    Installing : kernel-5.10.201-191.748.amzn2.x86_64           1/1
    Verifying  : kernel-5.10.201-191.748.amzn2.x86_64           1/1

    Installed:
    kernel.x86_64 0:5.10.201-191.748.amzn2
    ```

3. Download building tools.
    ```
    yum groupinstall "Development Tools" -y
    yum install -y libmount-devel libcap-devel gperf glib2-devel python3-pip
    pip3 install --user meson ninja jinja2
    export PATH=$PATH:~/.local/bin
    ```

4. Download and extract systemd source files.
    ```
    curl -OL https://github.com/systemd/systemd/archive/refs/tags/v254.tar.gz
    tar -xf v254.tar.gz
    cd systemd-254/
    ```

    The latest version of systemd is 254 at the time of writing. You can choose to install a different or more up to date version of systemd from [systemd's github release page](https://github.com/systemd/systemd/releases).

5. Build and install systemd.

    ```
    ./configure
    make -j $(nproc --all)
    make install -j $(nproc --all)
    systemctl --version
    ```

6. Rebuild `initramfs`.

    The initramfs needs to be rebuilt with the newly installed systemd modules. The modules must be built for the newly installed 5.10 kernel. You can rebuild the initramfs for this kernel using the `dracut` command, specifying the version of the kernel installed in step 1.
    ```
    dracut -f -v --kver 5.10.201-191.748.amzn2.x86_64
    ```

7. Reboot.

    ```
    reboot
    ```

After these steps, you would notice that cgroup2 is now enabled. You can confirm if this is the case by using the below command:

```
stat -fc %T /sys/fs/cgroup/
```

If the command returns `cgroup2fs`, cgroupv2 has been successfully enabled, which is the expected output after the above steps.


## Conclusion

The above steps can be used to enable cgroupv2 on Amazon Linux 2, at least until Amazon releases an updated systemd version, or until Amazon Linux 2 is sunset and we move on to the next version. In the meantime however, we will have to settle for manually upgrading systemd to get cgroupv2. 