作为 Java 开发者，你是否已经厌倦了 Spring Boot 容器化过程中的各种复杂配置和坑点？想要尝试更轻量、更高效的框架？那么 Solon 绝对值得你关注。今天我就带你实战 Solon 框架与 Docker 的集成，从环境准备到最终部署，全程避坑指南，让你 30 分钟内轻松搞定！

### 为什么选择 Solon + Docker？

在微服务架构盛行之下，应用容器化已成为标配。但传统的 Spring Boot 虽然功能强大，但在启动速度、内存占用和容器化体验上仍有优化空间。听一个老同事说，他们公司经常有1GB大小的 Spring Boot Jar 包。

### Solon 的优势：

* 启动速度极快：Solon 应用的启动时间通常是 Spring Boot 的 `1/10` 到 `1/5`
* 内存占用更小：基础镜像体积更小，运行时内存消耗更低。通常只有 Spring Boot 的 `1/10` 到 `1/2`
* 配置更简洁：Docker 集成配置简单明了，减少踩坑概率
* 原生支持容器化：从设计之初就考虑了云原生场景

### 环境准备：三步搞定基础配置

在开始之前，确保你的本地环境满足以下要求：

#### 1. 确认环境版本

* JDK 版本：Solon 支持 JDK 8+，推荐使用 JDK 11 或 17 以获得更好的容器化支持
* Docker 版本：Docker 20.10+，推荐使用 Docker Desktop 4.0+
* Maven 版本：Maven 3.6+，确保插件兼容性

快速验证命令：

```
java -version
docker -v
mvn -v
```

#### 2. 创建 Solon 项目

如果你还没有 Solon 项目，可以通过网页版生成器快速创建：

```
https://solon.noear.org/start/
```

### 核心步骤：Docker 集成实战

#### 1. 配置 Maven 插件

在 `pom.xml` 中添加 Docker 打包插件。这里我们使用经过验证的 `spotify` 插件：

```
<plugin>
    <groupId>com.spotifygroupId>
    <artifactId>docker-maven-pluginartifactId>
    <version>1.2.2version>
    <configuration>
        
        <imageName>solon-demoimageName>
        <imageTags>
            <imageTag>${project.version}imageTag>
            <imageTag>latestimageTag>
        imageTags>
        
        <baseImage>adoptopenjdk/openjdk11:jre-11.0.11_9-alpinebaseImage>
        
        <entryPoint>["java", "-jar", "/${project.build.finalName}.jar", "--server.port=8080", "--drift=1"]entryPoint>
        <resources>
            <resource>
                <targetPath>/targetPath>
                <directory>${project.build.directory}directory>
                <include>${project.build.finalName}.jarinclude>
            resource>
        resources>
    configuration>
plugin>
```

避坑提示：

* 使用 alpine 版本的 JDK 镜像可以显著减小镜像体积（如果不兼容，可以再换个别的）
* entryPoint 必须使用数组格式，确保参数传递正确
* 确保 finalName 与打包后的 jar 包名称一致
* 加上`--drift=1`表示当前环境ip会漂移的（如果有注册服务，当下线时要求不作健康检测）。这是 solon 对云原生的一种优化。

### 2.备选方案：使用 Dockerfile

如果你更喜欢传统的 Dockerfile 方式，可以在项目根目录创建 Dockerfile：

```
# 使用轻量级基础镜像
FROM adoptopenjdk/openjdk11:jre-11.0.11_9-alpine

# 设置工作目录
WORKDIR /app

# 复制 jar 文件
COPY target/solon-demo-1.0.0.jar app.jar

# 暴露端口（根据你的应用配置调整）
EXPOSE 8080

# 启动应用
ENTRYPOINT ["java", "-jar", "app.jar", "--server.port=8080", "--drift=1"]
```

然后在 pom.xml 中配置插件使用 Dockerfile：

```
<plugin>
    <groupId>com.spotifygroupId>
    <artifactId>docker-maven-pluginartifactId>
    <version>1.2.2version>
    <configuration>
        <imageName>solon-demoimageName>
        <dockerDirectory>${project.basedir}dockerDirectory>
        <resources>
            <resource>
                <targetPath>/targetPath>
                <directory>${project.build.directory}directory>
                <include>${project.build.finalName}.jarinclude>
            resource>
        resources>
    configuration>
plugin>
```

### 3. 构建和运行

构建 Docker 镜像：

```
# 先打包应用
mvn clean package

# 构建 Docker 镜像
mvn docker:build
```

构建成功后，验证镜像：

```
docker images | grep solon-demo
```

运行容器：

```
# 第一次运行
docker run -d -p 8080:8080 --name solon-app solon-demo

# 查看运行状态
docker ps | grep solon-app

# 查看日志
docker logs solon-app
```

容器管理命令：

```
# 停止容器
docker stop solon-app

# 重启容器
docker restart solon-app

# 删除容器
docker rm solon-app
```

### 进阶技巧：优化和部署

#### 1. 镜像标签管理和推送

为镜像打标签并推送到镜像仓库：

```
# 打标签
docker tag solon-demo:latest your-repo/solon-demo:1.0.0
docker tag solon-demo:latest your-repo/solon-demo:latest

# 推送到仓库
docker push your-repo/solon-demo:1.0.0
docker push your-repo/solon-demo:latest
```

#### 2. 生产环境配置

对于生产环境，建议添加健康检查和资源限制：

```
docker run -d \
  -p 8080:8080 \
  --name solon-app \
  --memory=512m \
  --cpus=1.0 \
  solon-demo
```

### 常见问题排查

#### 1. 容器启动后立即退出

* 检查应用启动日志：docker logs solon-app
* 确认 jar 包路径正确
* 验证端口是否被占用

#### 2. 应用无法访问

* 检查端口映射：docker ps 确认端口映射关系
* 验证防火墙设置
* 检查应用监听的地址（确保是 0.0.0.0 而不是 127.0.0.1）

#### 3. 镜像体积过大

* 使用 alpine 版本的基础镜像
* 多阶段构建去除构建依赖
* 使用 JRE 而不是完整的 JDK

### 总结

Solon 与 Docker 的集成相比传统框架更加轻量简洁，主要优势体现在：

* 配置简单：Maven 插件配置直观，减少出错概率
* 镜像小巧：基础镜像选择灵活，最终镜像体积更小
* 启动快速：容器启动速度更快，适合快速扩缩容

通过本文的实战指南，你应该能够在 30 分钟内完成 Solon 应用的 Docker 化。赶紧拿起你的 Solon 项目实践一下吧！如果在实践中遇到任何问题，欢迎在评论区交流讨论。

本博客参考[nuts坚果](https://tidati.com)。转载请注明出处！
