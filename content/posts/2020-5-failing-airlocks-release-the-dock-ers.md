+++
title = "Removing containers, volumes, and images in one CTRL-V, Return"
date = "2020-05-01"
tags = ["docker", "containers", "one-liner"]
+++

Sometimes you need Docker to fuck off.

```shell script
docker stop $(docker ps -a -q) && \
docker rm $(docker ps -a -q) && \
docker rmi $(docker images -a -q) && \
docker volume rm $(docker volume ls -q)
```

...this does the trick (usually).