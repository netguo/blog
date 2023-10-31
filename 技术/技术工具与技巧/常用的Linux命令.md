##### 在一个目录中查找包含指定文本的文件
```  
find ./ -type f -name "*.xml" |xargs grep "body.properties"
```
当脱离编译器，在命令行里这个显得尤为重要

##### 解压war包
jar xvf temp.war