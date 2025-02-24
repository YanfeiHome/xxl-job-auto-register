xxl-job是一款非常优秀的任务调度中间件，轻量级、使用简单、支持分布式等优点，让它广泛应用在我们的项目中，解决了不少定时任务的调度问题。

我们都知道，在使用过程中需要先到xxl-job的任务调度中心页面上，配置**执行器executor**和具体的**任务job**，这一过程如果项目中的定时任务数量不多还好说，如果任务多了的话还是挺费工夫的。

![图片](https://mmbiz.qpic.cn/mmbiz_png/zpom4BeZSicbjV0ibabs4rE1oPWkf7GuP9hJLKA5ic63aFvb8OwHszWJib1rBKrmKjdLJDNJE1OOUYqs0CUibB1wKVA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

假设项目中有上百个这样的定时任务，那么每个任务都需要走一遍绑定`jobHander`后端接口，填写`cron`表达式这个流程…

我就想问问，填多了谁能不迷糊？

于是出于功能优化（**偷懒**）这一动机，前几天我萌生了一个想法，有没有什么方法能够告别xxl-job的管理页面，能够让我不再需要到页面上去手动注册执行器和任务，实现让它们自动注册到调度中心呢。

## 分析

分析一下，其实我们要做的很简单，只要在项目启动时主动注册`executor`和各个`jobHandler`到调度中心就可以了，流程如下：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/zpom4BeZSicbjV0ibabs4rE1oPWkf7GuP9VFo1k0DuicJ5JkcRvWn3e5a114q3qJ83xJDjQDic278lNLneHF4wibiaUQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

有的小伙伴们可能要问了，我在页面上创建**执行器**的时候，不是有一个选项叫做**自动注册**吗，为什么我们这里还要自己添加新执行器？



其实这里有个误区，这里的自动注册指的是会根据项目中配置的`xxl.job.executor.appname`，将配置的机器地址自动注册到这个执行器的地址列表中。但是如果你之前没有手动创建过执行器，那么是不会给你自动添加一个新执行器到调度中心的。

既然有了想法咱们就直接开干，先到github上拉一份xxl-job的源码下来：

> https://github.com/xuxueli/xxl-job/https://github.com/xuxueli/xxl-job/

整个项目导入idea后，先看一下结构：



结合着文档和代码，先梳理一下各个模块都是干什么的：

- `xxl-job-admin`：任务调度中心，启动后就可以访问管理页面，进行执行器和任务的注册、以及任务调用等功能了
- `xxl-job-core`：公共依赖，项目中使用到xxl-job时要引入的依赖包
- `xxl-job-executor-samples`：执行示例，分别包含了springboot版本和不使用框架的版本

为了弄清楚注册和查询`executor`和`jobHandler`调用的是哪些接口，我们先从页面上去抓一个请求看看：

![图片](https://mmbiz.qpic.cn/mmbiz_png/zpom4BeZSicbjV0ibabs4rE1oPWkf7GuP9BcmTOJJGJV9W8oamw1fibP8BZBicwx9SwpDblEJkqkF0AtcicEAViboEoA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

好了，这样就能定位到`xxl-job-admin`模块中`/jobgroup/save`这个接口，接下来可以很容易地找到源码位置：

![图片](https://mmbiz.qpic.cn/mmbiz_png/zpom4BeZSicbjV0ibabs4rE1oPWkf7GuP95Ad4l17cxW6sKeMGxhhic4bsLK6Jo5XFs1ndNc152766LDeRNLNYadw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

按照这个思路，可以找到下面这几个关键接口：

- `/jobgroup/pageList`：执行器列表的条件查询
- `/jobgroup/save`：添加执行器
- `/jobinfo/pageList`：任务列表的条件查询
- `/jobinfo/add`：添加任务

但是如果直接调用这些接口，那么就会发现它会跳转到`xxl-job-admin`的的登录页面：



其实想想也明白，出于安全性考虑，调度中心的接口也不可能允许裸调的。那么再回头看一下刚才页面上的请求就会发现，它在`Headers`中添加了一条名为`XXL_JOB_LOGIN_IDENTITY`的`cookie`：



至于这条`cookie`，则是在通过用户名和密码调用调度中心的`/login`接口时返回的，在返回的`response`可以直接拿到。只要保存下来，并在之后每次请求时携带，就能够正常访问其他接口了。

到这里，我们需要的5个接口就基本准备齐了，接下来准备开始正式的改造工作。

## 改造

我们改造的目的是实现一个`starter`，以后只要引入这个`starter`就能实现`executor`和`jobHandler`的自动注册，要引入的关键依赖有下面两个：

```
<dependency>
    <groupId>com.xuxueli</groupId>
    <artifactId>xxl-job-core</artifactId>
    <version>2.3.0</version>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure</artifactId>
</dependency>
```

### 1、接口调用

在调用调度中心的接口前，先把`xxl-job-admin`模块中的`XxlJobInfo`和`XxlJobGroup`这两个类拿到我们的starter项目中，用于接收接口调用的结果。

#### 登录接口

创建一个`JobLoginService`，在调用业务接口前，需要通过登录接口获取`cookie`，并在获取到`cookie`后，缓存到本地的`Map`中。

```
private final Map<String,String> loginCookie=new HashMap<>();

public void login() {
    String url=adminAddresses+"/login";
    HttpResponse response = HttpRequest.post(url)
            .form("userName",username)
            .form("password",password)
            .execute();
    List<HttpCookie> cookies = response.getCookies();
    Optional<HttpCookie> cookieOpt = cookies.stream()
            .filter(cookie -> cookie.getName().equals("XXL_JOB_LOGIN_IDENTITY")).findFirst();
    if (!cookieOpt.isPresent())
        throw new RuntimeException("get xxl-job cookie error!");

    String value = cookieOpt.get().getValue();
    loginCookie.put("XXL_JOB_LOGIN_IDENTITY",value);
}
```

其他接口在调用时，直接从缓存中获取`cookie`，如果缓存中不存在则调用`/login`接口，为了避免这一过程失败，允许最多重试3次。

```
public String getCookie() {
    for (int i = 0; i < 3; i++) {
        String cookieStr = loginCookie.get("XXL_JOB_LOGIN_IDENTITY");
        if (cookieStr !=null) {
            return "XXL_JOB_LOGIN_IDENTITY="+cookieStr;
        }
        login();
    }
    throw new RuntimeException("get xxl-job cookie error!");
}
```

#### 执行器接口

创建一个`JobGroupService`，根据`appName`和执行器名称`title`查询执行器列表：

```
public List<XxlJobGroup> getJobGroup() {
    String url=adminAddresses+"/jobgroup/pageList";
    HttpResponse response = HttpRequest.post(url)
            .form("appname", appName)
            .form("title", title)
            .cookie(jobLoginService.getCookie())
            .execute();

    String body = response.body();
    JSONArray array = JSONUtil.parse(body).getByPath("data", JSONArray.class);
    List<XxlJobGroup> list = array.stream()
            .map(o -> JSONUtil.toBean((JSONObject) o, XxlJobGroup.class))
            .collect(Collectors.toList());
    return list;
}
```

我们在后面要根据配置文件中的`appName`和`title`判断当前执行器是否已经被注册到调度中心过，如果已经注册过那么则跳过，而`/jobgroup/pageList`接口是一个模糊查询接口，所以在查询列表的结果列表中，还需要再进行一次精确匹配。

```
public boolean preciselyCheck() {
    List<XxlJobGroup> jobGroup = getJobGroup();
    Optional<XxlJobGroup> has = jobGroup.stream()
            .filter(xxlJobGroup -> xxlJobGroup.getAppname().equals(appName)
                    && xxlJobGroup.getTitle().equals(title))
            .findAny();
    return has.isPresent();
}
```

注册新`executor`到调度中心：

```
public boolean autoRegisterGroup() {
    String url=adminAddresses+"/jobgroup/save";
    HttpResponse response = HttpRequest.post(url)
            .form("appname", appName)
            .form("title", title)
            .cookie(jobLoginService.getCookie())
            .execute();
    Object code = JSONUtil.parse(response.body()).getByPath("code");
    return code.equals(200);
}
```

#### 任务接口

创建一个`JobInfoService`，根据执行器`id`，`jobHandler`名称查询任务列表，和上面一样，也是模糊查询：

```
public List<XxlJobInfo> getJobInfo(Integer jobGroupId,String executorHandler) {
    String url=adminAddresses+"/jobinfo/pageList";
    HttpResponse response = HttpRequest.post(url)
            .form("jobGroup", jobGroupId)
            .form("executorHandler", executorHandler)
            .form("triggerStatus", -1)
            .cookie(jobLoginService.getCookie())
            .execute();

    String body = response.body();
    JSONArray array = JSONUtil.parse(body).getByPath("data", JSONArray.class);
    List<XxlJobInfo> list = array.stream()
            .map(o -> JSONUtil.toBean((JSONObject) o, XxlJobInfo.class))
            .collect(Collectors.toList());

    return list;
}
```

注册一个新任务，最终返回创建的新任务的`id`：

```
public Integer addJobInfo(XxlJobInfo xxlJobInfo) {
    String url=adminAddresses+"/jobinfo/add";
    Map<String, Object> paramMap = BeanUtil.beanToMap(xxlJobInfo);
    HttpResponse response = HttpRequest.post(url)
            .form(paramMap)
            .cookie(jobLoginService.getCookie())
            .execute();

    JSON json = JSONUtil.parse(response.body());
    Object code = json.getByPath("code");
    if (code.equals(200)){
        return Convert.toInt(json.getByPath("content"));
    }
    throw new RuntimeException("add jobInfo error!");
}
```

### 2、创建新注解

在创建任务时，必填字段除了执行器和`jobHandler`之外，还有**任务描述**、**负责人**、**Cron表达式**、**调度类型**、**运行模式**。在这里，我们默认调度类型为`CRON`、运行模式为`BEAN`，另外的3个字段的信息需要用户指定。

因此我们需要创建一个新注解`@XxlRegister`，来配合原生的`@XxlJob`注解进行使用，填写这几个字段的信息：

```
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface XxlRegister {
    String cron();
    String jobDesc() default "default jobDesc";
    String author() default "default Author";
    int triggerStatus() default 0;
}
```

最后，额外添加了一个`triggerStatus`属性，表示任务的默认调度状态，0为停止状态，1为运行状态。

### 3、自动注册核心

基本准备工作做完后，下面实现自动注册执行器和`jobHandler`的核心代码。核心类实现`ApplicationListener`接口，在接收到`ApplicationReadyEvent`事件后开始执行自动注册逻辑。

```
@Component
public class XxlJobAutoRegister implements ApplicationListener<ApplicationReadyEvent>, 
        ApplicationContextAware {
    private static final Log log =LogFactory.get();
    private ApplicationContext applicationContext;
    @Autowired
    private JobGroupService jobGroupService;
    @Autowired
    private JobInfoService jobInfoService;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext=applicationContext;
    }

    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        addJobGroup();//注册执行器
        addJobInfo();//注册任务
    }
}
```

自动注册执行器的代码非常简单，根据配置文件中的`appName`和`title`精确匹配查看调度中心是否已有执行器被注册过了，如果存在则跳过，不存在则新注册一个：

```
private void addJobGroup() {
    if (jobGroupService.preciselyCheck())
        return;

    if(jobGroupService.autoRegisterGroup())
        log.info("auto register xxl-job group success!");
}
```

自动注册任务的逻辑则相对复杂一些，需要完成：

- 通过`applicationContext`拿到spring容器中的所有bean，再拿到这些bean中所有添加了`@XxlJob`注解的方法
- 对上面获取到的方法进行检查，是否添加了我们自定义的`@XxlRegister`注解，如果没有则跳过，不进行自动注册
- 对同时添加了`@XxlJob`和`@XxlRegister`的方法，通过执行器id和`jobHandler`的值判断是否已经在调度中心注册过了，如果已存在则跳过
- 对于满足注解条件且没有注册过的`jobHandler`，调用接口注册到调度中心

具体代码如下：

```
private void addJobInfo() {
    List<XxlJobGroup> jobGroups = jobGroupService.getJobGroup();
    XxlJobGroup xxlJobGroup = jobGroups.get(0);

    String[] beanDefinitionNames = applicationContext.getBeanNamesForType(Object.class, false, true);
    for (String beanDefinitionName : beanDefinitionNames) {
        Object bean = applicationContext.getBean(beanDefinitionName);

        Map<Method, XxlJob> annotatedMethods  = MethodIntrospector.selectMethods(bean.getClass(),
                new MethodIntrospector.MetadataLookup<XxlJob>() {
                    @Override
                    public XxlJob inspect(Method method) {
                        return AnnotatedElementUtils.findMergedAnnotation(method, XxlJob.class);
                    }
                });
        for (Map.Entry<Method, XxlJob> methodXxlJobEntry : annotatedMethods.entrySet()) {
            Method executeMethod = methodXxlJobEntry.getKey();
            XxlJob xxlJob = methodXxlJobEntry.getValue();

            //自动注册
            if (executeMethod.isAnnotationPresent(XxlRegister.class)) {
                XxlRegister xxlRegister = executeMethod.getAnnotation(XxlRegister.class);
                List<XxlJobInfo> jobInfo = jobInfoService.getJobInfo(xxlJobGroup.getId(), xxlJob.value());
                if (!jobInfo.isEmpty()){
                    //因为是模糊查询，需要再判断一次
                    Optional<XxlJobInfo> first = jobInfo.stream()
                            .filter(xxlJobInfo -> xxlJobInfo.getExecutorHandler().equals(xxlJob.value()))
                            .findFirst();
                    if (first.isPresent())
                        continue;
                }

                XxlJobInfo xxlJobInfo = createXxlJobInfo(xxlJobGroup, xxlJob, xxlRegister);
                Integer jobInfoId = jobInfoService.addJobInfo(xxlJobInfo);
            }
        }
    }
}
```

### 4、自动装配

创建一个配置类，用于扫描`bean`：

```
@Configuration
@ComponentScan(basePackages = "com.xxl.job.plus.executor")
public class XxlJobPlusConfig {
}
```

将它添加到`META-INF/spring.factories`文件：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.xxl.job.plus.executor.config.XxlJobPlusConfig
```

到这里`starter`的编写就完成了，可以通过maven发布jar包到本地或者私服：

```
mvn clean install/deploy
```

## 测试

新建一个springboot项目，引入我们在上面打好的包：

```
<dependency>
    <groupId>com.cn.hydra</groupId>
    <artifactId>xxljob-autoregister-spring-boot-starter</artifactId>
    <version>0.0.1</version>
</dependency>
```

在`application.properties`中配置xxl-job的信息，首先是原生的配置内容：

```
xxl.job.admin.addresses=http://127.0.0.1:8080/xxl-job-admin
xxl.job.accessToken=default_token
xxl.job.executor.appname=xxl-job-executor-test
xxl.job.executor.address=
xxl.job.executor.ip=127.0.0.1
xxl.job.executor.port=9999
xxl.job.executor.logpath=/data/applogs/xxl-job/jobhandler
xxl.job.executor.logretentiondays=30
```

此外还要额外添加我们自己的starter要求的新配置内容：

```
# admin用户名
xxl.job.admin.username=admin
# admin 密码
xxl.job.admin.password=123456
# 执行器名称
xxl.job.executor.title=test-title
```

完成后在代码中配置一下`XxlJobSpringExecutor`，然后在测试接口上添加原生`@XxlJob`注解和我们自定义的`@XxlRegister`注解：

```
@XxlJob(value = "testJob")
@XxlRegister(cron = "0 0 0 * * ? *",
        author = "hydra",
        jobDesc = "测试job")
public void testJob(){
    System.out.println("#公众号：码农参上");
}


@XxlJob(value = "testJob222")
@XxlRegister(cron = "59 1-2 0 * * ?",
        triggerStatus = 1)
public void testJob2(){
    System.out.println("#作者：Hydra");
}

@XxlJob(value = "testJob444")
@XxlRegister(cron = "59 59 23 * * ?")
public void testJob4(){
    System.out.println("hello xxl job");
}
```

启动项目，可以看到执行器自动注册成功：

![图片](https://mmbiz.qpic.cn/mmbiz_png/zpom4BeZSicbjV0ibabs4rE1oPWkf7GuP9zvwvib6sInfIqPdDF8kKELkicHntCVIGVxprxulexKljlYEHUXp5xFNQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

再打开调度中心的任务管理页面，可以看到同时添加了两个注解的任务也已经自动完成了注册：

![图片](https://mmbiz.qpic.cn/mmbiz_png/zpom4BeZSicbjV0ibabs4rE1oPWkf7GuP9g1h1YyY5YIMoWNy2RRajibt4CNdic1guIwZGxSWEH10Bibnsu5DYLGeAg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

从页面上手动执行任务进行测试，可以执行成功：

![图片](https://mmbiz.qpic.cn/mmbiz_png/zpom4BeZSicbjV0ibabs4rE1oPWkf7GuP9GOsRUemIoSYgYElzVtarckExamCiaSHAwibdU99HGQREB9F9Pmw3k8fQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

到这里，starter的编写和测试过程就算基本完成了，项目中引入后，以后也能省出更多的时间来摸鱼学习了~

## xxljob-autoregister-spring-boot-starter

**********************************

自动注册xxl-job执行器以及任务

## 1、打包

```
mvn clean install
```

## 2、项目中引入

```xml
<dependency>
    <groupId>com.cn.hydra</groupId>
    <artifactId>xxljob-autoregister-spring-boot-starter</artifactId>
    <version>0.0.1</version>
</dependency>
```

## 3、配置

springboot项目配置文件application.properties：

```properties
server.port=8082

# 原生xxl-job配置
xxl.job.admin.addresses=http://127.0.0.1:8080/xxl-job-admin
xxl.job.accessToken=default_token
xxl.job.executor.appname=xxl-job-executor-test
xxl.job.executor.address=
xxl.job.executor.ip=127.0.0.1
xxl.job.executor.port=9999
xxl.job.executor.logpath=/data/applogs/xxl-job/jobhandler
xxl.job.executor.logretentiondays=30

# 新增配置项
# admin用户名
xxl.job.admin.username=admin 
# admin 密码
xxl.job.admin.password=123456
# 执行器名称
xxl.job.executor.title=Exe-Titl
```

`XxlJobSpringExecutor`参数配置与之前相同

## 4、添加注解
需要自动注册的方法添加注解`@XxlRegister`，不加则不会自动注册

```java
@Service
public class TestService {

    @XxlJob(value = "testJob")
    @XxlRegister(cron = "0 0 0 * * ? *",
            author = "hydra",
            jobDesc = "测试job")
    public void testJob(){
        System.out.println("#公众号：码农参上");
    }


    @XxlJob(value = "testJob222")
    @XxlRegister(cron = "59 1-2 0 * * ?",
            triggerStatus = 1)
    public void testJob2(){
        System.out.println("#作者：Hydra");
    }

    @XxlJob(value = "testJob444")
    @XxlRegister(cron = "59 59 23 * * ?")
    public void testJob4(){
        System.out.println("hello xxl job");
    }
}
```
