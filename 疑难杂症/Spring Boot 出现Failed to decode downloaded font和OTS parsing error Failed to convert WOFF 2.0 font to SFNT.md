#  Spring Boot 出现"Failed to decode downloaded font"和"OTS parsing error: Failed to convert WOFF 2.0 font to SFNT"



准确来讲，应该是maven项目使用Bootstrap时，出现

"**Failed to decode downloaded font**"和"**OTS parsing error: Failed to convert WOFF 2.0 font to SFNT**"

导致图标出不来的问题。

解决方案：

设置filter，font文件不需要filter，见下面示例：

```xml
  <resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>
                <excludes>
                    <exclude>static/fonts/**</exclude>
                </excludes>
            </resource>

            <resource>
                <directory>src/main/resources</directory>
                <filtering>false</filtering>
                <includes>
                    <include>static/fonts/**</include>
                </includes>
            </resource>
        </resources>
```

原因：

上面的xml里也写了，因为经过maven的filter，会破坏font文件的二进制文件格式，到时前台解析出错。