## 插件研究


```
<plugin>
<groupId>org.apache.maven.plugins</groupId>
<artifactId>maven-enforcer-plugin</artifactId>
<executions>
<execution>
<id>enforce-rules</id>
<goals>
<goal>enforce</goal>
</goals>
<configuration>
<rules>
<requireJavaVersion>
<version>[1.8,)</version>
</requireJavaVersion>
<requireProperty>
<property>main.basedir</property>
</requireProperty>
<requireProperty>
<property>project.organization.name</property>
</requireProperty>
<requireProperty>
<property>project.name</property>
</requireProperty>
<requireProperty>
<property>project.description</property>
</requireProperty>
</rules>
<fail>true</fail>
</configuration>
</execution>
</executions>
</plugin>
```



```
  <plugin>
                <groupId>org.basepom.maven</groupId>
                <artifactId>duplicate-finder-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <id>duplicate-dependencies</id>
                        <phase>validate</phase>
                        <goals>
                            <goal>check</goal>
                        </goals>
                        <configuration>
                            <ignoredResourcePatterns>
                                <ignoredResourcePattern>header\.txt$</ignoredResourcePattern>
                            </ignoredResourcePatterns>
                            <failBuildInCaseOfConflict>true</failBuildInCaseOfConflict>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
```











