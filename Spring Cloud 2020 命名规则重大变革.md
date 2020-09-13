> 白菜Java自习室 涵盖核心知识

Spring Cloud 版本重大变革，变更了版本号的命名方式。

**从 Spring Cloud 2020.0.0-M1 开始，Spring Cloud 废除了这种英国伦敦地铁站的命名方式，而使用了全新的 "日历化" 版本命名方式**。
![](https://user-gold-cdn.xitu.io/2020/7/13/17345b6b96251b57?w=812&h=404&f=png&s=40683)

下面是截取自官宣的一段内容：

https://spring.io/blog/2020/04/17/spring-cloud-2020-0-0-m1-released

> **Notable Changes in the 2020 Release Train**
> 
> We have changed our release train versioning scheme. We now follow Calendar Versioning or calver for short. We > > will follow the YYYY.MINOR.MICRO scheme where MINOR is an incrementing number that starts at zero each year. The MICRO segment corresponds to suffixes previously used: .0 is analogous to .RELEASE and .2 is analogous to .SR2. Pre-release suffixes will also change from using a . to a - for the separator, for example 2020.0.0-M1 and 2020.0.0-RC2. We will also stop prefixing snapshots with BUILD- – for example 2020.0.0-SNAPSHOT.
> 
> We will continue to use London Tube Station names for code names. The current codename is Ilford. These names will no longer be used in versions published to maven repositories.
> 
> Spring Cloud AWS and Spring Cloud GCP are no longer part of the release train. They will continue to be part of Hoxton as long as it is supported – at least thru June of 2021. Spring Cloud GCP will continue on as a separate project in https://github.com/GoogleCloudPlatform.
> 
> Initial milestones are based on Spring Boot 2.3.x but will shift to 2.4.x once that line has started.
> 
> The 2020.0 Release Train will be available on https://start.spring.io once development has started on the next feature release of Spring Boot (2.4.0). Please see Getting Started, below, for instructions on including this release in your project.
> 
> In all, a total of 183 issues, enhancements, bugs and pull requests were included in this release. See the GitHub project for details.

**什么是日历化版本？**

英文名称：**Calendar Versioning**

日历化版本不是基于任意的数字，而是基于项目的发布日期的版本控制约定，随着时间的推移，版本会越来越好。

这种基于日期的版本命名方式被称为 “日历化版本”（Calendar Versioning）， 或者可以简称 CalVer。

详细的介绍参考：

https://calver.org/

> **Versioning gets better with time.**
> 
> For maintainers, versioning allows us to specify precise dependencies within an ever-expanding ecosystem. For sellers and promoters, a project's version is a dynamic part of a brand. For all of us, versioning lets us reference the past while upgrading to the future.
> 
> Different projects use different systems for versioning, but common practices have emerged. For instance, point-separated numbers (e.g., 3.1.4) are all but given. Another common versioning pattern incorporates a time-based element, usually part of the release date.
> 
> This date-based approach has come to be called Calendar Versioning, or CalVer for short.

我们来看下 Spring Cloud 是如何开始使用日历化版本的。

Spring Cloud 使用了 YYYY.MINOR.MICRO 的命名规则：

* **YYYY**：表示 4 位年份；
* **MINOR**：代表一个递增的数字，每年以 0 开始递增；
* **MICRO**：代表版本号后缀，就和之前使用的 .0 类似于 .RELEASE 一样，.2 类似于 .SR2。
预发布版本的后缀分隔符也从 . 变更为 -，如：2020.0.0-M1 和 2020.0.0-RC2 命名所示。

同时，Spring Cloud 将停止给快照版本添加 BUILD- 前缀，如：2020.0.0-SNAPSHOT 命名所示。

但是，英国伦敦地铁站的命名没有彻底废除，Spring Cloud 将继续使用它作为版本代号，当前代号：Ilford，只是发布到 Maven 仓库的版本将不再使用这些名称。

最后就再来欣赏下 Maven 下的 Spring Cloud 新老版本号命名方式：

老版本命名：
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-dependencies</artifactId>
    <version>Hoxton.SR6</version>
    <type>pom</type>
    <scope>runtime</scope>
</dependency>
```
新版本命名：
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-dependencies</artifactId>
    <version>2020.0.0-M2</version>
    <type>pom</type>
    <scope>runtime</scope>
</dependency>
```
使用日历化版本命名方式，我个人觉得会更方便，可以更清楚的看出当前版本的年份，看到字母、纯数字方式的版本号都不知道自己多久没升级了。