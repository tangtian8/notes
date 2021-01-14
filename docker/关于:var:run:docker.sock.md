运行过Docker Hub的Docker镜像的话，会发现其中一些容器时需要挂载/var/run/docker.sock文件

它是Docker守护进程(Docker daemon)默认监听的Unix域套接字(Unix domain socket)，容器中的进程可以通过它与Docker守护进程进行通信。

