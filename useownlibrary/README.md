# 클라이언트 프로젝트 maven 설정
> maven-public이 maven-releases, maven-snapshots, maven-central 모두 포함하고 있다고 가정 

## 방법1) `~/.m2/settings.xml` 설정 : 표준화
```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                              http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <servers>
    <server>
      <id>nexus-public</id> <!-- id가 같을 필요는 없지만 헷갈리지 않기 위해 같게 씀 -->
      <username>admin</username>
      <password>관리자비밀번호</password>
    </server>
  </servers>

  <mirrors>
    <mirror>
      <id>nexus-public</id>
      <mirrorOf>*</mirrorOf>
      <url>http://<서버IP>:8081/repository/maven-public/</url>
    </mirror>
  </mirrors>
</settings>
```

### 방법2) 프로젝트 별 설정
- `프로젝트루트/.mvn/settings.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                              http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <servers>
    <server>
      <id>nexus-public</id>
      <username>admin</username>
      <password>admin</password>
    </server>
  </servers>

  <mirrors>
    <mirror>
      <id>nexus-public</id>
      <mirrorOf>*</mirrorOf> <!--maven 모든 원격 요청이 nexus-public으로 감 -->
      <url>http://<서버IP>:8081/repository/maven-public/</url>
    </mirror>
  </mirrors>
</settings>
```

- maven 실행 시 해당 settings.xml 설정
```bash
mvn clean install --settings .mvn/settings.xml
```

# 클라이언트 프로젝트 gradle 설정
- build.gradle
```gradle
repositories {
    maven {
        url "http://<서버IP>:8081/repository/maven-public/"
        allowInsecureProtocol = true // Nexus가 HTTPS 아니면 추가
        credentials {
            username = "admin"         // 익명 접근 막힌 경우만 필요
            password = "관리자비밀번호"
        }
    }
}
```