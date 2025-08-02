# Nexus Repository Manager 구성 가이드

## 1. Nexus 서버 구성
### 1) 설치 방식
> 개발/테스트용 : Docker
> 운영/프로덕션용 : On-Premise
- install/docker/README.md
- install/linux/README.md

### 2) 초기 설정
#### 관리자 로그인

* URL: `http://<서버IP>:8081`
* 초기 비밀번호:

  ```bash
  docker exec -it nexus cat /nexus-data/admin.password
  ```
- Docker를 사용하지 않는 경우, 설치 디렉토리 내 `nexus-data/admin.password`에서 확인

### 3) Repository 설계 및 생성
#### Release Repository
> 릴리즈 버전 정책이 적용된 Maven2 저장소 형식을 사용하며, 내부 릴리즈를 게시하는 저장소로 사용된다 
* Type: maven2 (hosted)
* Name: `maven-releases`
* Deployment Policy: Disable redeploy

#### Snapshot Repository
> 스냅샷 버전 정책이 적용된 Maven2 저장소 형식을 사용하며, 내부 개발 버전을 게시하는 용도이다
* Type: maven2 (hosted)
* Name: `maven-snapshots`
* Version policy: Snapshot

#### Proxy Repository (Maven Central 캐싱)
> 다른 원격 저장소에 연결된 저장소로 요청이 제일 먼저 도착하여 해당 구성요소가 로컬 콘텐츠에 있는지 확인한다 
> 프록시 저장소에서 로컬로 사용할 수 없는 경우 원격 저장소(외부 저장소, maven repository같은거)에서 해당 구성 요소를 찾는다 
* Type: maven2 (proxy)
* Name: `maven-central`
* Remote storage: [https://repo1.maven.org/maven2/](https://repo1.maven.org/maven2/)

#### Group Repository (통합)
* Type: maven2 (group)
* Name: `maven-public`
* Members 순서
  1. maven-releases
  2. maven-snapshots
  3. maven-central

## 2. 프로젝트 배포
### 1) pom.xml 설정
```xml
<distributionManagement>
  <repository>
    <id>nexus-releases</id>
    <url>http://<서버IP>:8081/repository/maven-releases/</url>
  </repository>
  <snapshotRepository>
    <id>nexus-snapshots</id>
    <url>http://<서버IP>:8081/repository/maven-snapshots/</url>
  </snapshotRepository>
</distributionManagement>
```

### 3-2. 배포 명령어

```bash
# 테스트 제외하고 배포
mvn clean deploy -DskipTests

# 테스트 포함 배포
mvn clean deploy
```

> 배포 규칙
>
> * `1.0.0-SNAPSHOT` → snapshot 레포지토리에 업로드
> * `1.0.0` → release 레포지토리에 업로드

---

## 4. 라이브러리 사용

### pom.xml에서 Nexus 참조

```xml
<repositories>
  <repository>
    <id>nexus-public</id>
    <url>http://<서버IP>:8081/repository/maven-public/</url>
  </repository>
</repositories>
```

---

## 5. 트러블슈팅

### 401 Unauthorized 오류

* `settings.xml`의 `<server><id>` 값과 `pom.xml`의 `<repository><id>` 값 일치 여부 확인
* Nexus 사용자 계정 권한 확인

### 배포 권한 오류

* Nexus UI에서 해당 사용자에 `nx-repository-admin-*-*` 권한 부여

### 네트워크 연결 오류

```bash
curl -u admin:password http://<서버IP>:8081/service/rest/v1/repositories
```

### Maven 로컬 캐시 문제

```bash
mvn dependency:purge-local-repository
```

---

## 6. Docker 여부와 관계없는 운영 주의사항

### 데이터 경로와 백업

* `/nexus-data` 또는 `sonatype-work/nexus3` 경로 필수 백업
* Release Repository는 redeploy 불가하므로 복구 대비 필요

### Release / Snapshot 분리

* `-SNAPSHOT` 버전은 snapshot 레포지토리로만 업로드
* Release와 Snapshot 분리하지 않으면 운영/개발 혼선

### Proxy / Group Repository

* Maven Central Proxy 활용해 사내 빌드 속도 및 안정성 확보
* Group Repository(`maven-public`)로 Proxy + Hosted를 묶어 단일 URL 제공

### 보안 / 계정 관리

* 기본 admin 계정은 비밀번호 변경 후 사용
* CI/CD 전용 계정 최소 권한 발급
* LDAP/SSO 연동 고려

### 포트 및 HTTPS

* 기본 포트 8081 충돌 시 변경
* 운영 환경에서는 Nginx/Traefik으로 HTTPS 적용 권장

### 리소스 및 성능

* 메모리 최소 2GB, 권장 4GB
* SSD 스토리지 사용 시 인덱싱/검색 성능 개선
* JVM 옵션으로 `-Xmx`, `-XX:MaxDirectMemorySize` 조정

### 운영 정책

* Release Redeploy 금지
* Snapshot 자동 정리(예: 30일 이후 삭제)
* CI/CD 파이프라인에서 태깅-릴리즈-스냅샷 전환 자동화

---

## 7. 버전 마이그레이션 및 업그레이드

### 7-1. Nexus 버전 업그레이드

#### Docker 환경에서 업그레이드

```bash
# 현재 컨테이너 중지
docker stop nexus

# 데이터 백업
docker cp nexus:/nexus-data ./nexus-data-backup

# 새 버전으로 컨테이너 실행
docker run -d -p 8081:8081 \
  --name nexus \
  -v nexus-data:/nexus-data \
  sonatype/nexus3:latest
```

#### On-Premise 환경에서 업그레이드

```bash
# 서비스 중지
sudo systemctl stop nexus

# 현재 설치 디렉토리 백업
sudo cp -r /opt/nexus /opt/nexus-backup
sudo cp -r /opt/sonatype-work /opt/sonatype-work-backup

# 새 버전 다운로드 및 설치
wget https://download.sonatype.com/nexus/3/latest-unix.tar.gz
tar -xzf latest-unix.tar.gz
sudo mv nexus-* /opt/nexus-new

# 기존 설정 마이그레이션
sudo cp -r /opt/sonatype-work/nexus3 /opt/nexus-new/
sudo systemctl start nexus
```

### 7-2. 데이터 마이그레이션

#### Repository 설정 백업

```bash
# Nexus API를 통한 설정 백업
curl -u admin:password \
  -X GET "http://localhost:8081/service/rest/v1/repositories" \
  -H "accept: application/json" > repositories-backup.json

# Blob store 설정 백업
curl -u admin:password \
  -X GET "http://localhost:8081/service/rest/v1/blobstores" \
  -H "accept: application/json" > blobstores-backup.json
```

#### 아티팩트 마이그레이션

```bash
# 특정 repository의 모든 아티팩트 목록
curl -u admin:password \
  -X GET "http://localhost:8081/service/rest/v1/search?repository=maven-releases" \
  -H "accept: application/json" > artifacts-list.json

# 아티팩트 다운로드 스크립트 예시
for artifact in $(cat artifacts-list.json | jq -r '.items[].downloadUrl'); do
  wget --user=admin --password=password "$artifact" -P ./artifacts-backup/
done
```

### 7-3. 버전별 호환성 체크

#### Nexus 3.x 주요 변경사항

```bash
# Java 버전 요구사항 확인
java -version

# 최소 Java 8, 권장 Java 11+
# Nexus 3.30+ 부터는 Java 8 지원 중단
```

#### Maven 호환성 확인

```bash
# Maven 버전 확인
mvn -version

# settings.xml 문법 검증
mvn help:effective-settings
```

### 7-4. 롤백 절차

#### Docker 환경 롤백

```bash
# 새 컨테이너 중지
docker stop nexus
docker rm nexus

# 백업 데이터로 롤백
docker run -d -p 8081:8081 \
  --name nexus \
  -v ./nexus-data-backup:/nexus-data \
  sonatype/nexus3:3.37.3  # 이전 안정 버전
```

#### On-Premise 환경 롤백

```bash
# 서비스 중지
sudo systemctl stop nexus

# 백업에서 복원
sudo rm -rf /opt/nexus
sudo mv /opt/nexus-backup /opt/nexus
sudo rm -rf /opt/sonatype-work
sudo mv /opt/sonatype-work-backup /opt/sonatype-work

# 서비스 재시작
sudo systemctl start nexus
```

---

## 8. 유용한 명령어

```bash
# 특정 아티팩트만 배포
mvn deploy:deploy-file \
  -DgroupId=com.example \
  -DartifactId=my-app \
  -Dversion=1.0.0 \
  -Dpackaging=jar \
  -Dfile=target/my-app-1.0.0.jar \
  -DrepositoryId=nexus \
  -Durl=http://<서버IP>:8081/repository/maven-releases/

# 의존성 트리 확인
mvn dependency:tree

# Nexus에서 다운로드되는지 확인
mvn dependency:resolve -U
```
