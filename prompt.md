# AWS 하이브리드 세션 스토어 구축 가이드

AWS 도쿄 리전에서 ElastiCache Valkey와 DynamoDB를 사용한 하이브리드 세션 스토어를 구축해주세요.

## 요구사항

### 인프라 구성
- ElastiCache Valkey: 6개 노드 (3 primary + 3 replica), cache.t3.micro, TLS 비활성화
- DynamoDB: session-store 테이블, Pay-per-request, TTL 활성화
- EC2: Amazon Linux 2023, t3.micro, DynamoDB 접근 권한 포함
- VPC: 전용 네트워크, 적절한 보안 그룹 설정

### 애플리케이션 기능
- valkey-glide 라이브러리 사용
- 하이브리드 세션 스토어 구현:
  1. 세션 생성: ElastiCache + DynamoDB 동시 저장
  2. 세션 조회: ElastiCache 우선, 실패시 DynamoDB 조회 후 캐시 복원
  3. TTL 관리: 양쪽 스토어 동기화
  4. 고가용성: 한쪽 스토어 장애시에도 서비스 지속

### 테스트 시나리오
- 50명 사용자 데이터로 성능 테스트
- 캐시 히트/미스 시뮬레이션
- 스토리지별 통계 조회
- 자동 캐시 복원 검증

### 배포 방식
- CloudFormation 템플릿으로 인프라 자동 배포
- 자동 배포 스크립트 포함
- EC2에서 직접 실행 가능한 Python 애플리케이션

### 파일 구성
1. elasticache-valkey-stack.yaml - CloudFormation 템플릿
2. hybrid_session_store.py - 하이브리드 세션 스토어 구현
3. create_dynamodb_table.py - DynamoDB 테이블 생성
4. deploy.sh - 자동 배포 스크립트
5. requirements.txt - Python 의존성

모든 리소스는 테스트 완료 후 쉽게 정리할 수 있도록 구성해주세요.

---

## ⚠️ 안정적인 배포를 위한 체크리스트

### 📋 **사전 준비 체크리스트**
- [ ] **AWS 문서 확인**: CloudFormation 속성명, 데이터 타입 제한사항 사전 확인
- [ ] **로컬 테스트 환경**: DynamoDB 로컬 또는 실제 테이블로 기능 검증
- [ ] **시뮬레이션 모드**: ElastiCache 연결 실패 시 대체 로직 준비
- [ ] **디버깅 도구**: 헬스 체크, 연결 테스트 스크립트 미리 작성

### 🚀 **배포 전략 체크리스트**
- [ ] **User Data 단순화**: 복잡한 스크립트 대신 단순한 설치 명령어 사용
- [ ] **단계별 배포**: 인프라 → 기본 기능 → 고급 기능 순서로 진행
- [ ] **헬스 체크 필수**: `/health` 엔드포인트로 서비스 상태 확인 가능
- [ ] **로그 수집**: User Data 실행 로그, 애플리케이션 로그 확인 방법 준비

### 🌐 **네트워크 고려사항**
- [ ] **VPC 제한 인지**: ElastiCache는 VPC 내부에서만 접근 가능
- [ ] **Session Manager 설정**: SSH 연결 실패 시 대안 접근 방법
- [ ] **보안 그룹 검증**: 필요한 포트(22, 8000, 6379) 허용 규칙 확인
- [ ] **로컬 대안**: 로컬 Redis 컨테이너 또는 시뮬레이션 모드 준비

### 💾 **데이터 호환성 체크리스트**
- [ ] **DynamoDB 타입**: float 대신 int 사용 (`time.time()` → `int(time.time())`)
- [ ] **JSON 직렬화**: 모든 데이터가 JSON 직렬화 가능한지 확인
- [ ] **TTL 설정**: DynamoDB TTL 속성명과 형식 정확히 사용
- [ ] **속성명 정확성**: `Description` vs `ReplicationGroupDescription` 등 정확한 속성명 사용

### 🔧 **실수 방지 가이드**

#### 1. **EC2 User Data 스크립트**
```bash
# ✅ 권장: 단순한 스크립트
#!/bin/bash
yum update -y
yum install -y python3 python3-pip
pip3 install boto3 valkey flask
# 간단한 테스트 스크립트만 포함

# ❌ 피해야 할: 복잡한 스크립트
# 200줄 이상의 복잡한 애플리케이션 코드 포함
```

#### 2. **CloudFormation 속성명**
```yaml
# ✅ 정확한 속성명
ElastiCacheReplicationGroup:
  Properties:
    ReplicationGroupDescription: "설명"

# ❌ 잘못된 속성명
ElastiCacheReplicationGroup:
  Properties:
    Description: "설명"  # 오류 발생
```

#### 3. **연결 실패 대응**
```python
# ✅ 시뮬레이션 모드 준비
try:
    self.valkey_client.ping()
    self.valkey_available = True
except:
    print("시뮬레이션 모드로 전환")
    self.valkey_available = False
```

#### 4. **데이터 타입 주의**
```python
# ✅ DynamoDB 호환 타입
'timestamp': int(time.time())
'ttl': int(time.time()) + 3600

# ❌ DynamoDB 비호환 타입
'timestamp': time.time()  # float 타입 오류
```

### 🎯 **핵심 원칙**
1. **단순함 우선**: 복잡한 구현보다 단순하고 안정적인 접근
2. **문서 확인**: AWS 공식 문서 확인 후 구현
3. **단계별 검증**: 각 단계별로 기능 검증 후 다음 단계 진행
4. **대안 준비**: 연결 실패, 서비스 오류 시 대체 방안 미리 준비
5. **로컬 우선**: 로컬에서 기능 검증 후 AWS 배포
