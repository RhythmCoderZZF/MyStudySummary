### 配置

#### 仓库镜像、gradle下载被墙

1. 阿里云中央仓库镜像

   ```groovy
   maven { url 'http://maven.aliyun.com/nexus/content/groups/public/'}
   maven { url'https://maven.aliyun.com/repository/public/' }
   maven { url'https://maven.aliyun.com/repository/google/' }
   maven { url'https://maven.aliyun.com/repository/jcenter/' }
   maven { url'https://maven.aliyun.com/repository/central/' }
   ```

   

2. gradle

   浏览器直接访问 https://services.gradle.org/distributions/gradle-6.6.1-all.zip 

3. Gradle Sync failed

   命令行 ./gradlew --debug

