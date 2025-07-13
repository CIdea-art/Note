**Q:** Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?

- 服务没启动

    ```shell
    service docker start
    ```

**Q:** permission denied

- 加入docker用户组中

  ```bash
  sudo gpasswd -a $USER docker
  # 更新用户组
  newgrp docker
  ```

  
