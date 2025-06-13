+++
date = 2020-07-25
draft = false
title = 'The NFS Buffer Cache'
weight = 10
[params]
  author = 'Jenade Moodley'
showTableOfContents = true
categories = ['Linux']
tags = ['NFS']
+++


If you are using an NFS compatible Distributed File System (such as a Samba server or EFS), you may notice a slight delay when writing files to the file system, more specifically, when you create a new file from one of the NFS clients, it may take a couple of seconds longer to appear on another client, even though it is already available on the original client.


This behaviour can be explained by the NFS buffer cache. By default, when you are writing data to a file in Linux, it is first written in memory. This is appropriately called a buffer, i.e. it can be described as a smaller and faster storage area, which sits in front of the actual storage device. This is to improve performance as memory is much faster than a hard disk or solid state drive. The same is true for NFS, where writes are first written to the NFS buffer, before being flushed (or synced) to the NFS storage. This improves performance on the client performing the writes as the clients do not have to wait for the data to be synced to the NFS disk on every write, allowing more writes to be processed.

{{< figure
    src="img/buffer-cache.png"
    alt="NFS Buffer Cache diagram"
    caption="[Image by Howard Hwa, Brian Kennedy, Ken Myers, and Robert Wei](https://read.seas.harvard.edu/~kohler/class/05f-osp/notes/lec18.html)"
    >}}

However, if your application is heavily dependent on consistency, i.e. it needs to be available on another client as soon as the write is made, this buffer will not be ideal for you. Instead, you can then opt-in to allow writes to be written directly to the server, bypassing the cache. This can be controlled by the `sync` NFS mount option. By default, the NFS client will use the `async` mount option unless specified otherwise. `async` refers to asynchronous, where data is flushed asynchronously to the main server. If you switch to using the `sync` mount option, data will be flushed to the server immediately, which allows for greater consistency but at a great performance cost. Referring to the [NFS man pages](https://linux.die.net/man/5/nfs):


<style>
code { text-wrap: wrap; }
}
</style>
```bash {class="my-class" id="my-codeblock" tabWidth=2}
The sync mount option
    The NFS client treats the sync mount option differently than some other file systems (refer to mount(8) for a description of the generic sync and async mount options). If neither sync nor async is specified (or if the async option is specified), the NFS client delays sending application writes to the server until any of these events occur:

       Memory pressure forces reclamation of system memory resources.

       An application flushes file data explicitly with sync(2), msync(2), or fsync(3).

       An application closes a file with close(2).

       The file is locked/unlocked via fcntl(2).

    In other words, under normal circumstances, data written by an application may not immediately appear on the server that hosts the file.

    If the sync option is specified on a mount point, any system call that writes data to files on that mount point causes that data to be flushed to the server before the system call returns control to user space. This provides greater data cache coherence among clients, but at a significant performance cost.

    Applications can use the O_SYNC open flag to force application writes to individual files to go to the server immediately without the use of the sync mount option.
```