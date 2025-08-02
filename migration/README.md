# 서버 마이그레이션

## 데이터 경로
- 도커 
  - `/nexus-data`
- 온프레미스 
  - `/sonatype-work/nexus3` : 모든 아티팩트, 설정, 계정 정보

## 마이그레이션 절차

### 1. Nexus 서비스 중지 후 백업

```bash
# 서비스 중지
sudo systemctl stop nexus
# 또는 도커의 경우
docker stop nexus

# 데이터 백업
tar czvf nexus-data-backup.tar.gz /opt/sonatype-work/nexus3
```

### 2. 새 서버에 Nexus 설치

```bash
# 백업 파일 전송
scp nexus-data-backup.tar.gz user@new-server:/opt/

# 새 서버에서 압축 해제
tar xzvf nexus-data-backup.tar.gz -C /opt/sonatype-work/
```

### 3. 경로 권한 확인

```bash
sudo chown -R nexus:nexus /opt/sonatype-work/nexus3
```

### 4. 서비스 시작 및 검증
```bash
sudo systemctl start nexus
docker start nexus
```

---

# 버전 업그레이드

## 업그레이드 절차

### 1. 현재 버전 확인

```bash
cat /opt/nexus/lib/support/nexus-coreapp-*.jar | grep Implementation-Version
```

### 2. 서비스 중지 및 데이터 백업

```bash
# 서비스 중지
sudo systemctl stop nexus

# 데이터 백업
tar czvf nexus-upgrade-backup-$(date +%Y%m%d).tar.gz /opt/sonatype-work/nexus3
```

### 3. 새 버전 다운로드 및 교체

```bash
# 새 버전 다운로드
wget https://download.sonatype.com/nexus/3/nexus-3.xx.x-unix.tar.gz
tar -xvzf nexus-3.xx.x-unix.tar.gz

# 기존 nexus 폴더 교체
mv /opt/nexus /opt/nexus-old
mv nexus-3.xx.x /opt/nexus
chown -R nexus:nexus /opt/nexus
```

### 4. 서비스 시작 및 마이그레이션 모니터링

```bash
# 서비스 시작
sudo systemctl start nexus

# DB 마이그레이션 로그 모니터링 (자동 실행)
tail -f /opt/sonatype-work/nexus3/log/nexus.log
```