

# Spring Boot



##  一、主程序类 入口类

​		@**SpringBootApplication**应用标注在某个类上说明这个类是SpringBoot的主配置类；通过被标注类来启动SpringBoot

​	

```
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(
    excludeFilters = {@Filter(
    type = FilterType.CUSTOM,
    classes = {TypeExcludeFilter.class}
), @Filter(
    type = FilterType.CUSTOM,
    classes = {AutoConfigurationExcludeFilter.class}
)}
)
```

​	@**SpringBootConfiguration**: Spring Boot 的配置类

​			标注在某个类上，表示这是一个Spring Boot的配置类

​				@**Configuration**： 配置类上标注这个注解

​							配置类   配置文件：配置类也是容器中的一个组件：@Component



@**EnableAutoConfiguration**：开启自动配置功能；

```java
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class})
```

​		@**AutoConfigurationPackage** ：自动配置包

​				@Import({Registrar.class})

​					Spring的底层注解@import给容器导入一个组件；

​					Registrar 作用：==将主配置类（@**SpringBootApplication**标注的类）所在的包及所有子包里所有的组件扫描到Spring容器==

​		@**Import**({AutoConfigurationImportSelector.class})

​				给容器导入组件

​					**AutoConfigurationImportSelector**：导入组件选择器

​					将所有需要导入的组件以全类名的方式返回，这些组件就会被添加到容器中；

​					会给容器导入自动配置类（XXXAutoConfiguration）：目的是给容器导入这个场景所需的组件，并配置好这些组件

SpringFactoriesLoader.loadFactoryNames(EnableAutoConfiguration.class, this.getBeanClassLoader());

==Spring Boot在启动时从类路径下的META-INF/spring.factories中获取EnableAutoConfiguration指定的值，将这些值作为配置类导入到容器中，自动配置类就生效了，即可进行自动配置工作==

## 二、使用Spring Initializer创建Spring Boot项目

​			默认生成的Sring Boot项目：

- 主程序已经生成好了，只需关心代码逻辑
- resource文件夹目录结构
  - static存放静态资源
  - template：保存所有的模板页面（Spring Boot的jar包是使用的嵌入式的Tomcat，默认不支持JSP页面），可以使用模板引擎（freemarker、thymeleaf）
  - application.yml :Spring Boot 的应用配置文件
  

## 三、Spring Boot配置文件

​	SpringBoot使用一个全局的配置文件，配置文件名是固定的

​	application*.yml

​	application*.yaml*

​	application*.properties

配置文件的作用：修改SpringBoot自动配置的默认值

​	@ConfigurationProperties(prefix = "person")它可以让开发者将整个配置文件，映射到对象中

​	prefix = "person" 指定配置文件的属性名

只有组件是Spring容器的组件，才能使用容器提供的功能，需与@Component 配合使用

