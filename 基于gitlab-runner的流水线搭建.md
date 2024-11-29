

# 基于gitlab-runner的流水线搭建

------

> [!NOTE]
>
> 为了保证宿主机环境不被管道工作污染，同时保证多台边端可以快速搭建自动构建环境，采用容器化方式部署gitlab-runner。

> [!WARNING]
>
> 根据实验结果，容器化部署的gitlab-runner的**shell**执行器功能受限，具体表现为无法在gitlab-runner容器内使用docker运行时命令，所以采用**docker**执行器做为gitlab-runner的构建驱动。

![image-20241129095303059](D:\vscodeProject\know-house\Typora\typora-user-images\image-20241129095303059.png)

## 容器化部署gitlab-runner

使用docker-compose可以更快捷的启动、停止、以及删除容器。

展示compose.yaml文件：

```yaml
name: cicd

services:
  gitlab-runner:
    image: bitnami/gitlab-runner:latest
    restart: always
    volumes:
      - ./gitlab-runner-region:/home/gitlab-runner
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - common

networks:
  common:
```

这是我gitlab-runner服务的路径。

```
root@firefly:/opt/cicd# tree -a
.
├── compose.yaml
└── gitlab-runner-region
    └── .gitlab-runner
        ├── config.toml
        └── .runner_system_id

2 directories, 3 files
```

容器化部署gitlab-runner时，需要通过配置config.toml文件来注册**runner**。

配置文件中**url**是gitlab仓库地址；**token**是gitlab仓库密钥签名；**executor**必须设置为**docker**。

展示config.toml文件：

```toml
concurrent = 10
log_level = "info"

[[runners]]
  name = "lced"
  url = "URL"
  token = "TOKEN"
  executor = "docker"
  [runners.docker]
    ...
```

接下来需要对**runners.docker**进行配置。根据实验结果以及**docker-in-docker**概念，大致为两种配置方案。

- 配置1：

流水线管道工作镜像（指**docker:latest**镜像）共享宿主机的**docker套接字文件**，仅使用工作镜像中的docker客户端工具与宿主机的docker服务通信。

配置文件中的**image**是流水线管道使用的默认镜像（不指定具体镜像时使用），这里使用gitlab推荐的**ruby：2.7**镜像。

同时需要禁用容器特权，防止容器获得宿主机的所有权限。

还需要将宿主机的/var/run/docker.sock文件挂载到流水线管道工作容器中。

展示config.toml文件：

```toml
concurrent = 10
log_level = "info"

[[runners]]
  name = "lced"
  url = "URL"
  token = "TOKEN"
  executor = "docker"
  [runners.docker]
    image = "ruby:2.7"
    privileged = false
    volumes = ["/var/run/docker.sock:/var/run/docker.sock"]
    
```

- 配置2：

流水线管道工作镜像（指**docker:latest**镜像）不共享宿主机的**docker套接字文件**，在工作容器内部自动生成/var/run/docker.sock文件。

需要启动**容器特权**，docker套接字引擎的创建需要宿主机**privileged**权限。<u>这会给宿主机带来一定的风险，建议工作镜像从私有仓库下载来保证安全性。</u>

不再需要/var/run/docker.sock文件挂载，因为工作镜像不依赖宿主机的**docker套接字文件**。

```toml
concurrent = 10
log_level = "info"
  
[[runners]]
  name = "lced"
  url = "URL"
  token = "TOKEN"
  executor = "docker"
  [runners.docker]
    tls_verify = false
    image = "ruby:2.7"
    privileged = true   
```

- 配置3：

深入了解docker-in-docker概念以及gitlab-runner运行docker机制后，对**配置2**进行优化，更符合官方推荐的配置。

在docke客户端与docker套接字文件通信时，添加tls验证。

docker:latest镜像默认支持tls验证，后续在.gitlab-ci.yaml中不用在使用**DOCKER_TLS_CERTDIR**环境变量切换校验方式。

```toml
concurrent = 10
log_level = "info"
  
[[runners]]
  name = "lced"
  url = "URL"
  token = "TOKEN"
  executor = "docker"
  [runners.docker]
    tls_verify = true
    image = "ruby:2.7"
    privileged = true
    volumes = ["/certs/client"]
```



## 编写流水线构建脚本.gitlab-ci.yml

流水线构建脚本符合yaml文档格式，通过gitlab指定的语法定义工作流管道。

- 实验1：

此实验在gitlab-runner**配置2**的基础上编写流水线脚本。

展示.gitlab-ci.yml文件：

```yaml
stages:
  - build
  - upload

variables:
  VERSION: "0.1.0"
  VERSION_INDEX: "0"
  ENVIRONMENT: "dev"
  ARCHITECTURE: "arm64"
  PROJECT_NAME: "PROJECT_NAME"
  HARBOR_USER: "HARBOR_USER"
  HARBOR_PASSWORD: "HARBOR_PASSWORD"
  HARBOR_HOST: "HARBOR_HOST"
  HARBOR_IMAGE: "${HARBOR_HOST}/lced/${PROJECT_NAME}:${VERSION}-${VERSION_INDEX}-${ENVIRONMENT}-${ARCHITECTURE}"
  DOCKER_TLS_CERTDIR: ""

build:
  stage: build
  image: docker:latest
  services:
    - docker:latest
  script:
    - env
    - docker images
    - docker info
  tags:
    - lced

upload:
  stage: upload
  image: docker:latest
  services:
    - docker:latest
  script:
    - echo "rmi ${HARBOR_IMAGE}"
  tags:
    - lced


```

展示工作流运行时的容器：

![image-20241129221209666](D:\vscodeProject\know-house\Typora\typora-user-images\image-20241129221209666.png)

runner-s9txxwglq-project-54-concurrent-1-db5155408a493253-build是工作容器（即**image**定义的镜像），负责执行管道中定义的命令。

runner-s9txxwglq-project-54-concurrent-1-db5155408a493253-docker-0是辅助容器 （即**services**定义的镜像），负责启动docker引擎的守护进程。

> [!IMPORTANT]
>
> 困扰到我的问题：
>
> 既然docker:latest镜像即存在docker客户端工具，又存在docker引擎守护进程，为什么必须使用**services**定义的辅助容器？
>
> 获得到的答案：
>
> 因为工作容器会被gitlab-runner重写**容器入口命令**参数，导致工作容器无法启动docker引擎守护进程。

runner-s9txxwglq-project-54-concurrent-1-db5155408a493253-build的启动命令：

![image-20241129222715142](D:\vscodeProject\know-house\Typora\typora-user-images\image-20241129222715142.png)

runner-s9txxwglq-project-54-concurrent-1-db5155408a493253-docker-0的启动命令：

![image-20241129222929018](D:\vscodeProject\know-house\Typora\typora-user-images\image-20241129222929018.png)

> [!WARNING]
>
> 报错1：ERROR: error during connect: Head "http://docker:2375/_ping": dial tcp: lookup docker on 192.168.2.1:53: no such host.
>
> 解释：未启动辅助容器（即**services**定义的镜像）.
>
> 报错2：Cannot connect to the Docker daemon at tcp://docker:2375. Is the docker daemon running?
>
> 解释：辅助容器启动，但是监听在了tcp://0.0.0.0:2376，所有工作容器无法连接辅助容器的docker引擎。

展示辅助容器进程：

```
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 docker-init -- dockerd --host=unix:///var/run/docker.sock --host=tcp://0.0.0.0:2376 --tlsverify --tlscacert /certs/server/ca.pem --tlscert /certs/server/cert.pem --tlskey /certs/server/key.pem
   52 root      0:00 dockerd --host=unix:///var/run/docker.sock --host=tcp://0.0.0.0:2376 --tlsverify --tlscacert /certs/server/ca.pem --tlscert /certs/server/cert.pem --tlskey /certs/server/key.pem
   63 root      0:00 containerd --config /var/run/docker/containerd/containerd.toml
  234 root      0:00 /bin/sh
  247 root      0:00 ps aux
```

展示摘录自docker:latest镜像的dockerd-entrypoint.sh文件：

```shell
if [ "$#" -eq 0 ] || [ "${1#-}" != "$1" ]; then
	# set "dockerSocket" to the default "--host" *unix socket* value (for both standard or rootless)
	uid="$(id -u)"
	if [ "$uid" = '0' ]; then
		dockerSocket='unix:///var/run/docker.sock'
	else
		# if we're not root, we must be trying to run rootless
		: "${XDG_RUNTIME_DIR:=/run/user/$uid}"
		dockerSocket="unix://$XDG_RUNTIME_DIR/docker.sock"
	fi
	case "${DOCKER_HOST:-}" in
		unix://*)
			dockerSocket="$DOCKER_HOST"
			;;
	esac

	# add our default arguments
	if [ -n "${DOCKER_TLS_CERTDIR:-}" ]; then
		_tls_generate_certs "$DOCKER_TLS_CERTDIR"
		# generate certs and use TLS if requested/possible (default in 19.03+)
		set -- dockerd \
			--host="$dockerSocket" \
			--host=tcp://0.0.0.0:2376 \
			--tlsverify \
			--tlscacert "$DOCKER_TLS_CERTDIR/server/ca.pem" \
			--tlscert "$DOCKER_TLS_CERTDIR/server/cert.pem" \
			--tlskey "$DOCKER_TLS_CERTDIR/server/key.pem" \
			"$@"
		DOCKERD_ROOTLESS_ROOTLESSKIT_FLAGS="${DOCKERD_ROOTLESS_ROOTLESSKIT_FLAGS:-} -p 0.0.0.0:2376:2376/tcp"
	else
		# TLS disabled (-e DOCKER_TLS_CERTDIR='') or missing certs
		set -- dockerd \
			--host="$dockerSocket" \
			--host=tcp://0.0.0.0:2375 \
			"$@"
		DOCKERD_ROOTLESS_ROOTLESSKIT_FLAGS="${DOCKERD_ROOTLESS_ROOTLESSKIT_FLAGS:-} -p 0.0.0.0:2375:2375/tcp"
	fi
fi
```

通过分析：

默认的**dockerd**命令参数为--host=unix:///var/run/docker.sock --host=tcp://0.0.0.0:2375。

当开启TLS检验时，docker引擎启动在--host=tcp://0.0.0.0:2376，否则启动在--host=tcp://0.0.0.0:2375。

- 实验2：

此实验在gitlab-runner**配置3**的基础上编写流水线脚本。

展示.gitlab-ci.yml文件：

```
stages:
  - build
  - upload

variables:
  VERSION: "0.1.0"
  VERSION_INDEX: "0"
  ENVIRONMENT: "dev"
  ARCHITECTURE: "arm64"
  PROJECT_NAME: "PROJECT_NAME"
  HARBOR_USER: "HARBOR_USER"
  HARBOR_PASSWORD: "HARBOR_PASSWORD"
  HARBOR_HOST: "HARBOR_HOST"
  HARBOR_IMAGE: "${HARBOR_HOST}/lced/${PROJECT_NAME}:${VERSION}-${VERSION_INDEX}-${ENVIRONMENT}-${ARCHITECTURE}"

build:
  stage: build
  image: docker:latest
  services:
    - docker:latest
  script:
    - env
    - docker images
    - docker info
  tags:
    - lced

upload:
  stage: upload
  image: docker:latest
  services:
    - docker:latest
  script:
    - echo "rmi ${HARBOR_IMAGE}"
  tags:
    - lced
```

docker引擎启动在--host=tcp://0.0.0.0:2376，docker客户端通过默认环境变量**DOCKER_HOST=tcp://docker:2376**直接连接。

- 实验3：

此实验在gitlab-runner**配置1**的基础上编写流水线脚本。

展示.gitlab-ci.yml文件：

```
stages:
  - build
  - upload

variables:
  VERSION: "0.1.0"
  VERSION_INDEX: "0"
  ENVIRONMENT: "dev"
  ARCHITECTURE: "arm64"
  PROJECT_NAME: "PROJECT_NAME"
  HARBOR_USER: "HARBOR_USER"
  HARBOR_PASSWORD: "HARBOR_PASSWORD"
  HARBOR_HOST: "HARBOR_HOST"
  HARBOR_IMAGE: "${HARBOR_HOST}/lced/${PROJECT_NAME}:${VERSION}-${VERSION_INDEX}-${ENVIRONMENT}-${ARCHITECTURE}"

build:
  stage: build
  image: docker:latest
  script:
    - docker images
    - docker info
  tags:
    - lced

upload:
  stage: upload
  image: docker:latest
  script:
    - echo "rmi ${HARBOR_IMAGE}"
  tags:
    - lced
```

当使用**配置1**时，挂载宿主机的docker引擎，不需要再指定辅助容器（辅助容器生成docker.sock会冲突失败）。

```
Service container logs:
2024-11-29T16:19:35.404697638Z Certificate request self-signature ok
2024-11-29T16:19:35.404783386Z subject=CN=docker:dind server
2024-11-29T16:19:35.492196007Z /certs/server/cert.pem: OK
2024-11-29T16:19:37.094597405Z Certificate request self-signature ok
2024-11-29T16:19:37.094653112Z subject=CN=docker:dind client
2024-11-29T16:19:37.179333934Z /certs/client/cert.pem: OK
2024-11-29T16:19:37.189798082Z cat: can't open '/proc/net/arp_tables_names': No such file or directory
2024-11-29T16:19:37.195392980Z iptables v1.8.10 (nf_tables)
2024-11-29T16:19:37.319013759Z time="2024-11-29T16:19:37.318144323Z" level=info msg="Starting up"
2024-11-29T16:19:37.323614099Z failed to load listeners: can't create unix socket /var/run/docker.sock: device or resource busy
```



---



# 参考

- docker-in-docker：https://asciinema.org/a/378669
- config.toml语法：https://docs.gitlab.com/runner/configuration/advanced-configuration.html
- .gitlab-ci.yml语法：https://docs.gitlab.com/ee/ci/yaml
- 使用docker构建docker镜像：https://docs.gitlab.com/ee/ci/docker/using_docker_build.html
- docker执行程序：https://docs.gitlab.com/runner/executors/docker.html
- docker:latest镜像构建：https://github.com/docker-library/docker/tree/9095b12e6b5eb72689fb2c15f76403ce35ce04f7/27

  

  
