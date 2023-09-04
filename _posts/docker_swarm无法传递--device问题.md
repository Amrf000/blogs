宿主Ubuntu 20，docker镜像也是ubuntu，采用https://github.com/jenkinsci/docker-swarm-plugin插件，容器内使用unionfuse-fs命令报错fuse: device not found, try 'modprobe fuse' first，需要传递--device /dev/fuse以运行容器，但是swarm engine并不支持；

## 解决方案：

### 1. 解除Ubuntu 宿主docker apparmor限制

```
mv /sbin/apparmor_parser /sbin/apparmor_parser1
```

 
### 2. 测试docker device逃逸后执行unionfs-fuse测试命令测试成功   

```
docker run -it --rm --cap-add=SYS_ADMIN -v /dev/fuse:/dev/fuse -v /tmp:/tmp dockers镜像:v1.1

cd /tmp/dev/docker/$(cat /proc/self/cgroup | grep docker | head -1 | sed 's/.*\/docker\/\(.*\)/\1/g') && echo a > devices.allow
```   
-----

```
mkdir test1

touch test1/t.txt

mkdir test2

unionfs-fuse -o cow "test1=RO" test2

cd test2

ls -al
```

### 3. 由于docker-swarm-plugin插件没有传递cap-add的参数，而实际查看docker /services/create接口，是可以传递的，所以需要修改插件，patch如下

```
From fa78ab315c0a4cb46dc4ef7729cfd0293d2e8ba3 Mon Sep 17 00:00:00 2001

From:  <lyw@xm.com>

Date: Sat, 12 Aug 2023 19:00:11 +0800

Subject: [PATCH] Add support for Custom Conatiner Args

---

 .../docker/swarm/DockerSwarmAgentTemplate.java | 12 ++++++++++--

 .../docker/swarm/DockerSwarmComputerLauncher.java | 14 ++++++++------

 .../swarm/docker/api/containers/ContainerSpec.java | 6 ++++--

 .../swarm/docker/api/service/ServiceSpec.java | 4 ++--

 .../docker/swarm/docker/api/task/TaskTemplate.java | 4 ++--

 .../swarm/DockerSwarmAgentTemplate/config.jelly | 3 +++

 6 files changed, 29 insertions(+), 14 deletions(-)

diff --git a/src/main/java/org/jenkinsci/plugins/docker/swarm/DockerSwarmAgentTemplate.java b/src/main/java/org/jenkinsci/plugins/docker/swarm/DockerSwarmAgentTemplate.java

index 53be6a6..be6eee5 100644

--- a/src/main/java/org/jenkinsci/plugins/docker/swarm/DockerSwarmAgentTemplate.java

+++ b/src/main/java/org/jenkinsci/plugins/docker/swarm/DockerSwarmAgentTemplate.java

@@ -37,6 +37,7 @@ public class DockerSwarmAgentTemplate implements Describable<DockerSwarmAgentTem

 private boolean osWindows;

 private String unixCommand;

 private String windowsCommand;

+ private String capabilityAdd;

 private String user;

 private String workingDir;

 private String metadata;

@@ -58,7 +59,7 @@ public class DockerSwarmAgentTemplate implements Describable<DockerSwarmAgentTem



 @DataBoundConstructor

 public DockerSwarmAgentTemplate(final String image, final String hostBinds, final String hostNamedPipes, final String dnsIps,

- final String dnsSearchDomains, final String unixCommand,final String windowsCommand, final String user, final String workingDir,

+ final String dnsSearchDomains, final String unixCommand,final String windowsCommand,final String capabilityAdd,final String user, final String workingDir,

 final String hosts, final String metadata, final String secrets, final String configs, final String label, final String cacheDir,

 final String tmpfsDir, final String envVars, final long limitsNanoCPUs, final long limitsMemoryBytes,

 final long reservationsNanoCPUs, final long reservationsMemoryBytes, String portBinds, final boolean osWindows,

@@ -71,6 +72,7 @@ public class DockerSwarmAgentTemplate implements Describable<DockerSwarmAgentTem

 this.dnsSearchDomains = dnsSearchDomains;

 this.unixCommand = unixCommand;

 this.windowsCommand = windowsCommand;

+ this.capabilityAdd = capabilityAdd;

 this.user = user;

 this.workingDir = workingDir;

 this.hosts = hosts;

@@ -154,7 +156,9 @@ public class DockerSwarmAgentTemplate implements Describable<DockerSwarmAgentTem

 public String[] getWindowsCommandConfig() {

 return StringUtils.isEmpty(this.windowsCommand) ? new String[] {} : this.windowsCommand.split("[\\r\\n\\s]+");

 }

-

+ public String[] getCapabilityAddConfig() {

+ return StringUtils.isEmpty(this.capabilityAdd) ? new String[] {} : this.capabilityAdd.split("[\\r\\n]+");

+ }

 public String[] getWindowsCommandConfig(String windowsCommand) {

 return StringUtils.isEmpty(windowsCommand) ? new String[] {} : windowsCommand.split("[\\r\\n\\s]+");

 }

@@ -266,6 +270,10 @@ public class DockerSwarmAgentTemplate implements Describable<DockerSwarmAgentTem

 return unixCommand;

 }



+ public String getCapabilityAdd() {

+ return capabilityAdd;

+ }

+

 public String getWindowsCommand() {

 return windowsCommand;

 }

diff --git a/src/main/java/org/jenkinsci/plugins/docker/swarm/DockerSwarmComputerLauncher.java b/src/main/java/org/jenkinsci/plugins/docker/swarm/DockerSwarmComputerLauncher.java

index b883765..a5a498e 100755

--- a/src/main/java/org/jenkinsci/plugins/docker/swarm/DockerSwarmComputerLauncher.java

+++ b/src/main/java/org/jenkinsci/plugins/docker/swarm/DockerSwarmComputerLauncher.java

@@ -112,11 +112,13 @@ public class DockerSwarmComputerLauncher extends JNLPLauncher {

 }

 }



- launchContainer(finalCommand.toArray(new String[0]), configuration, envVars, dockerSwarmAgentTemplate.getWorkingDir(),

+ launchContainer(finalCommand.toArray(new String[0]), dockerSwarmAgentTemplate.getCapabilityAddConfig(),

+ configuration, envVars, dockerSwarmAgentTemplate.getWorkingDir(),

 dockerSwarmAgentTemplate.getUser(), dockerSwarmAgentTemplate, listener, computer,

 dockerSwarmAgentTemplate.getHostsConfig());

 } else {

- launchContainer(dockerSwarmAgentTemplate.getUnixCommandConfig(), configuration, envVars,

+ launchContainer(dockerSwarmAgentTemplate.getUnixCommandConfig(), dockerSwarmAgentTemplate.getCapabilityAddConfig(),

+ configuration, envVars,

 dockerSwarmAgentTemplate.getWorkingDir(), dockerSwarmAgentTemplate.getUser(),

 dockerSwarmAgentTemplate, listener, computer, dockerSwarmAgentTemplate.getHostsConfig());

 }

@@ -134,11 +136,11 @@ public class DockerSwarmComputerLauncher extends JNLPLauncher {

 }

 }



- public void launchContainer(String[] commands, DockerSwarmCloud configuration, String[] envVars, String dir,

+ public void launchContainer(String[] commands, String[] capabilityAdd,DockerSwarmCloud configuration, String[] envVars, String dir,

 String user, DockerSwarmAgentTemplate dockerSwarmAgentTemplate, TaskListener listener,

 DockerSwarmComputer computer, String[] hosts) throws IOException {

 DockerSwarmPlugin swarmPlugin = Jenkins.getInstance().getPlugin(DockerSwarmPlugin.class);

- ServiceSpec crReq = createCreateServiceRequest(commands, configuration, envVars, dir, user,

+ ServiceSpec crReq = createCreateServiceRequest(commands, capabilityAdd, configuration, envVars, dir, user,

 dockerSwarmAgentTemplate, computer, hosts);



 setLimitsAndReservations(dockerSwarmAgentTemplate, crReq);

@@ -178,10 +180,10 @@ public class DockerSwarmComputerLauncher extends JNLPLauncher {

 dockerSwarmAgentTemplate.getPlacementArchitecture(), dockerSwarmAgentTemplate.getPlacementOperatingSystem());

 }



- private ServiceSpec createCreateServiceRequest(String[] commands, DockerSwarmCloud configuration, String[] envVars,

+ private ServiceSpec createCreateServiceRequest(String[] commands, String[] capabilityAdd, DockerSwarmCloud configuration, String[] envVars,

 String dir, String user, DockerSwarmAgentTemplate dockerSwarmAgentTemplate, DockerSwarmComputer computer,

 String[] hosts) throws IOException {

- ServiceSpec crReq = new ServiceSpec(computer.getName(), dockerSwarmAgentTemplate.getImage(), commands, envVars,

+ ServiceSpec crReq = new ServiceSpec(computer.getName(), dockerSwarmAgentTemplate.getImage(), commands, capabilityAdd, envVars,

 dir, user, hosts);

 return crReq;

 }

diff --git a/src/main/java/org/jenkinsci/plugins/docker/swarm/docker/api/containers/ContainerSpec.java b/src/main/java/org/jenkinsci/plugins/docker/swarm/docker/api/containers/ContainerSpec.java

index d9554cb..b5387a0 100755

--- a/src/main/java/org/jenkinsci/plugins/docker/swarm/docker/api/containers/ContainerSpec.java

+++ b/src/main/java/org/jenkinsci/plugins/docker/swarm/docker/api/containers/ContainerSpec.java

@@ -7,6 +7,7 @@ public class ContainerSpec {



 public final String Image;

 public final String[] Command;

+ public final String[] CapabilityAdd;

 public final String[] Env;

 public final String Dir;

 public final String User;

@@ -17,12 +18,13 @@ public class ContainerSpec {

 public List<Config> Configs = new ArrayList<>();



 public ContainerSpec() {

- this(null, null, null, null, null, null);

+ this(null, null, null, null, null, null, null);

 }



- public ContainerSpec(String image, String[] cmd, String[] env, String dir, String user, String[] hosts) {

+ public ContainerSpec(String image, String[] cmd, String[] capabilityAdd, String[] env, String dir, String user, String[] hosts) {

 this.Image = image;

 this.Command = cmd;

+ this.CapabilityAdd= capabilityAdd;

 this.Env = env;

 this.Dir = dir;

 this.User = user;

diff --git a/src/main/java/org/jenkinsci/plugins/docker/swarm/docker/api/service/ServiceSpec.java b/src/main/java/org/jenkinsci/plugins/docker/swarm/docker/api/service/ServiceSpec.java

index c9c984f..58d32cf 100755

--- a/src/main/java/org/jenkinsci/plugins/docker/swarm/docker/api/service/ServiceSpec.java

+++ b/src/main/java/org/jenkinsci/plugins/docker/swarm/docker/api/service/ServiceSpec.java

@@ -28,11 +28,11 @@ public class ServiceSpec extends ApiRequest {



 public List<Network> Networks = new ArrayList<>();



- public ServiceSpec(String name, String Image, String[] Cmd, String[] Env, String Dir, String User, String[] Hosts)

+ public ServiceSpec(String name, String Image, String[] Cmd, String[] capabilityAdd, String[] Env, String Dir, String User, String[] Hosts)

 throws IOException {

 super(HttpMethod.POST, "/services/create", CreateServiceResponse.class, ResponseType.CLASS);

 this.Name = name;

- this.TaskTemplate = new TaskTemplate(Image, Cmd, Env, Dir, User, Hosts);

+ this.TaskTemplate = new TaskTemplate(Image, Cmd, capabilityAdd, Env, Dir, User, Hosts);

 this.EndpointSpec = new EndpointSpec();

 }



diff --git a/src/main/java/org/jenkinsci/plugins/docker/swarm/docker/api/task/TaskTemplate.java b/src/main/java/org/jenkinsci/plugins/docker/swarm/docker/api/task/TaskTemplate.java

index 962bf36..2efa04b 100755

--- a/src/main/java/org/jenkinsci/plugins/docker/swarm/docker/api/task/TaskTemplate.java

+++ b/src/main/java/org/jenkinsci/plugins/docker/swarm/docker/api/task/TaskTemplate.java

@@ -15,8 +15,8 @@ public class TaskTemplate {

 // for reading from api

 }



- public TaskTemplate(String image, String[] cmd, String[] env, String dir, String user, String[] hosts) {

- this.ContainerSpec = new ContainerSpec(image, cmd, env, dir, user, hosts);

+ public TaskTemplate(String image, String[] cmd, String[] capabilityAdd, String[] env, String dir, String user, String[] hosts) {

+ this.ContainerSpec = new ContainerSpec(image, cmd, capabilityAdd, env, dir, user, hosts);

 }



 public void setPlacement(String[] placementConstraints, String architecture, String operatingSystem) {

diff --git a/src/main/resources/org/jenkinsci/plugins/docker/swarm/DockerSwarmAgentTemplate/config.jelly b/src/main/resources/org/jenkinsci/plugins/docker/swarm/DockerSwarmAgentTemplate/config.jelly

index 1ae2b31..7f647f2 100755

--- a/src/main/resources/org/jenkinsci/plugins/docker/swarm/DockerSwarmAgentTemplate/config.jelly

+++ b/src/main/resources/org/jenkinsci/plugins/docker/swarm/DockerSwarmAgentTemplate/config.jelly

@@ -15,6 +15,9 @@

 <f:entry title="Windows Command" field="windowsCommand">

 <f:textbox value="${dockerSwarmAgentTemplate.windowsCommand}" default="powershell.exe &amp; { Invoke-WebRequest -TimeoutSec 20 -OutFile agent.jar %DOCKER_SWARM_PLUGIN_JENKINS_AGENT_JAR_URL%; if($?) { java -jar agent.jar -jnlpUrl %DOCKER_SWARM_PLUGIN_JENKINS_AGENT_JNLP_URL% -secret %DOCKER_SWARM_PLUGIN_JENKINS_AGENT_SECRET% -noReconnect } }"/>

 </f:entry>

+ <f:entry title="CapabilityAdd" field="capabilityAdd">

+ <f:textarea value="${dockerSwarmAgentTemplate.capabilityAdd}" default=""/>

+ </f:entry>

 <f:entry title="Working Directory" field="workingDir">

 <f:textbox value="${dockerSwarmAgentTemplate.workingDir}" default="/tmp"/>

 </f:entry>

--

2.38.1.windows.1
```



