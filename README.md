# MyBatisGenerator-Tool

- [我的简书](http://www.jianshu.com/u/b88081164fe8)
- [我的博客](https://maimieng.com)

> 在 Intellij IDEA 中结合 Gradle 使用 MyBatis Generator 逆向生成代码

- Info:
 - JDK 1.8
 - Gradle 2.14
 - Intellij IDEA 2016.3

## 前言

由于 IDEA 的教程较少，且 MyBatis Generator 不支持 Gradle 直接运行，因此这次是在自己折腾项目过程中，根据一些参考资料加上自己的实践得出的结论，并附上相应的 Demo 可供自己未来参考，也与大家分享。

本文的 Demo 也可以当作工具直接导入 IDEA，加上自己的数据库信息即可生成代码。

## 创建项目

详细的创建步骤可以参考[使用 Gradle 构建 Struts 2 Web 应用](http://www.jianshu.com/p/228a4c5e64be)中「新建 Gradle Web 项目」一节即可。当创建完毕，需要等待 Gradle 联网构建，由于国内网络因素，可能需要稍作等待。当构建完成，目录结构应如下图一致：

![Gradle 构建完成](http://7xkam0.com1.z0.glb.clouddn.com/blog/gradle_mybatis_generator_01.png)

## 配置依赖

这里需要使用 MyBatis Generator，MySQL 驱动，以及 MyBatis Mapper。由于代码生成单独运行即可，不需要参与到整个项目的编译，因此在 build.gradle 中添加配置：

```groovy
configurations {
    mybatisGenerator
}
```

在 [Maven Repository (http://mvnrepository.com/)](http://mvnrepository.com/) 分别搜索依赖：

![org.mybatis.generator](http://7xkam0.com1.z0.glb.clouddn.com/blog/gradle_mybatis_generator_02.png)

![mysql-connector-java](http://7xkam0.com1.z0.glb.clouddn.com/blog/gradle_mybatis_generator_03.png)

![tk.mybatis:mapper](http://7xkam0.com1.z0.glb.clouddn.com/blog/gradle_mybatis_generator_04.png)

依赖的版本并不是局限于某个特定版本，可以选择适时相应的更新版本。添加到 build.gradle 的 dependencies 中（注意前面需将「compile group:」改为 「mybatisGenerator」）：

```groovy
dependencies {
    testCompile group: 'junit', name: 'junit', version: '4.11'

    mybatisGenerator 'org.mybatis.generator:mybatis-generator-core:1.3.5'
    mybatisGenerator 'mysql:mysql-connector-java:5.1.40'
    mybatisGenerator 'tk.mybatis:mapper:3.3.9'
}
```

## 设置数据库信息

在 resources 下，新建 mybatis 文件夹，并新建 config.properties 和 generatorConfig.xml，文件结构如下：

![配置文件目录](http://7xkam0.com1.z0.glb.clouddn.com/blog/gradle_mybatis_generator_05.png)

在 config.properties 中配置数据库和要生成的 Java 类的包：

```
# JDBC 驱动类名
jdbc.driverClassName=com.mysql.jdbc.Driver
# JDBC URL: jdbc:mysql:// + 数据库主机地址 + 端口号 + 数据库名
jdbc.url=jdbc:mysql://
# JDBC 用户名及密码
jdbc.username=
jdbc.password=

# 生成实体类所在的包
package.model=com.maimieng.model
# 生成 mapper 类所在的包
package.mapper=com.maimieng.mapper
# 生成 mapper xml 文件所在的包，默认存储在 resources 目录下
package.xml=mybatis
```

## 设置生成代码的配置文件

在 generatorConfig.xml 中配置数据库表信息，可以参考官方的文档（附在文末）来进行配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
    <context id="Mysql" targetRuntime="MyBatis3Simple" defaultModelType="flat">
        <commentGenerator>
            <property name="suppressAllComments" value="true"></property>
            <property name="suppressDate" value="true"></property>
            <property name="javaFileEncoding" value="utf-8"/>
        </commentGenerator>

        <jdbcConnection driverClass="${driverClass}"
                        connectionURL="${connectionURL}"
                        userId="${userId}"
                        password="${password}">
        </jdbcConnection>

        <javaTypeResolver>
            <property name="forceBigDecimals" value="false"/>
        </javaTypeResolver>

        <javaModelGenerator targetPackage="${modelPackage}" targetProject="${src_main_java}">
            <property name="enableSubPackages" value="true"></property>
            <property name="trimStrings" value="true"></property>
        </javaModelGenerator>

        <sqlMapGenerator targetPackage="${sqlMapperPackage}" targetProject="${src_main_resources}">
            <property name="enableSubPackages" value="true"></property>
        </sqlMapGenerator>

        <javaClientGenerator targetPackage="${mapperPackage}" targetProject="${src_main_java}" type="ANNOTATEDMAPPER">
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>

        <!-- sql占位符，表示所有的表 -->
        <table tableName="%">
            <generatedKey column="epa_id" sqlStatement="Mysql" identity="true" />
        </table>
    </context>
</generatorConfiguration>
```

## Gradle 设置 Ant Task

由于 MyBatis Generator 尚不支持 Gradle，所以只能使用 Gradle 来执行 Ant Task，达到相同的效果。

build.gradle:

```groovy
def getDbProperties = {
    def properties = new Properties()
    file("src/main/resources/mybatis/jdbc.properties").withInputStream { inputStream ->
        properties.load(inputStream)
    }
    properties
}

task mybatisGenerate << {
    def properties = getDbProperties()
    ant.properties['targetProject'] = projectDir.path
    ant.properties['driverClass'] = properties.getProperty("jdbc.driverClassName")
    ant.properties['connectionURL'] = properties.getProperty("jdbc.url")
    ant.properties['userId'] = properties.getProperty("jdbc.username")
    ant.properties['password'] = properties.getProperty("jdbc.password")
    ant.properties['src_main_java'] = sourceSets.main.java.srcDirs[0].path
    ant.properties['src_main_resources'] = sourceSets.main.resources.srcDirs[0].path
    ant.properties['modelPackage'] = properties.getProperty("package.model")
    ant.properties['mapperPackage'] = properties.getProperty("package.mapper")
    ant.properties['sqlMapperPackage'] = properties.getProperty("package.xml")
    ant.taskdef(
            name: 'mbgenerator',
            classname: 'org.mybatis.generator.ant.GeneratorAntTask',
            classpath: configurations.mybatisGenerator.asPath
    )
    ant.mbgenerator(overwrite: true,
            configfile: 'src/main/resources/mybatis/generatorConfig.xml', verbose: true) {
        propertyset {
            propertyref(name: 'targetProject')
            propertyref(name: 'userId')
            propertyref(name: 'driverClass')
            propertyref(name: 'connectionURL')
            propertyref(name: 'password')
            propertyref(name: 'src_main_java')
            propertyref(name: 'src_main_resources')
            propertyref(name: 'modelPackage')
            propertyref(name: 'mapperPackage')
            propertyref(name: 'sqlMapperPackage')
        }
    }
}
```

配置好 build.gradle，就可以在 IDEA 的 Gradle 菜单中找到新的 Task：

![mybatisGenerate](http://7xkam0.com1.z0.glb.clouddn.com/blog/gradle_mybatis_generator_06.png)

如果这里没有，可能是 Gradle 没有 build，重新刷新即可：

![Refresh all Gradle projects](http://7xkam0.com1.z0.glb.clouddn.com/blog/gradle_mybatis_generator_07.png)

双击「mybatisGenerate」即可运行，如果配置正确即可运行成功：

![成功生成代码](http://7xkam0.com1.z0.glb.clouddn.com/blog/gradle_mybatis_generator_08.png)

注：这里我在 generatorConfig.xml 配置选择的是按注解映射，因此没有 xml 文件生成。

## 总结

这次的 Demo 您可以在 [https://github.com/kingcos/MyBatisGenerator-Tool](https://github.com/kingcos/MyBatisGenerator-Tool) 下载查看。Demo 本身也可作为工具来生成代码。由于目前正在完善一个 Android + Java Web 的小系统，因此未来可能还会更新 Spring Boot、Druid、Swagger 的配置脚手架。

## 参考资料

- [MyBatis Generator](http://www.mybatis.org/generator/index.html)
- [abel533/Mapper](https://github.com/abel533/Mapper)
- [使用 Mapper 专用的 MyBatis Generator 插件](http://git.oschina.net/free/Mapper/blob/master/wiki/mapper3/7.UseMBG.md)
