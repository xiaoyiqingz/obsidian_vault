1. 创建工程  
`mvn archetype:generate -DgroupId=com.test -DartifactId=MvTest  -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false`

* -**DgroupId**: 组织名，公司网址的反写 + 项目名称
- **-DartifactId**: 项目名-模块名
- **-DarchetypeArtifactId**: 指定 ArchetypeId，`maven-archetype-quickstart`，创建一个简单的 Java 应用
- **-DinteractiveMode**: 是否使用交互模式


3. 直接运行mvn工程
	2.1 在 `pom.xml` 中加入
```java
<project>
...
	   <build>
        <plugins>
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>exec-maven-plugin</artifactId>
                <version>1.2.1</version>
                <configuration>
                    <mainClass>com.test.App</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>
...
</project>
```
	2.2 执行 `mvn clean compile exec:java`