# 배포 트러블슈팅 모음

> **작성일**: 2025-11-02  
> **프로젝트**: SignBell

---

## 트러블슈팅 1: Jenkins EC2 CPU 100% 및 응답 불가

**작성자**: 신동준

---

### 1. 문제 현상 (Problem)

> Jenkins 서버가 응답하지 않고 EC2 인스턴스 CPU 사용률이 100%에 도달

* **문제 1**: Jenkins 웹 페이지 접속 불가 (타임아웃)
* **문제 2**: EC2 SSH 접속 불가 또는 매우 느림
* **문제 3**: AWS 콘솔에서 CPU 사용률 100% 지속

<br>

**[CloudWatch 메트릭]**

```
CPUUtilization: 100%
MemoryUtilization: 95%
SwapUsage: High
```

---

### 2. 원인 분석 (Analysis)

> t3.small 인스턴스에서 Frontend, Backend, AI Server를 동시에 빌드하면서 메모리 부족 발생

* **원인 1: 인스턴스 스펙 부족**
    - t3.small (2GB RAM)로 3개 프로젝트 동시 빌드
    - Gradle, npm, Docker 빌드가 각각 500MB 이상 메모리 사용
    - 총 메모리 요구량이 인스턴스 용량 초과

* **원인 2: 스왑 메모리 과다 사용**
    - 물리 메모리 부족으로 스왑 메모리 사용
    - 디스크 I/O 증가로 CPU 사용률 급증
    - 시스템 전체 응답 속도 저하

---

### 3. 해결 방안 (Solution)

> 인스턴스 타입을 t3.medium으로 업그레이드하고 빌드 메모리 제한 설정

* **해결 방안 1: 인스턴스 타입 업그레이드**
    - t3.small (2GB RAM) → t3.medium (4GB RAM)
    - AWS 콘솔에서 인스턴스 중지 후 타입 변경
    - 비용 증가: $30/월 → $60/월

* **해결 방안 2: Gradle 메모리 제한**
    - `backend/gradle.properties` 파일 생성
  ```properties
  org.gradle.jvmargs=-Xmx512m -XX:MaxMetaspaceSize=256m
  org.gradle.daemon=false
  ```

* **해결 방안 3: npm 빌드 메모리 제한**
    - Jenkinsfile에서 Node.js 메모리 제한
  ```groovy
  sh 'NODE_OPTIONS="--max-old-space-size=512" npm run build'
  ```

---

### 4. 교훈 (Lessons Learned)

> 빌드 서버 리소스는 여유있게 확보하고 메모리 사용량을 모니터링해야 함

* **교훈 1: 리소스 요구사항 사전 계산**
    - 각 빌드 프로세스의 메모리 사용량 측정
    - 동시 빌드 시나리오 고려
    - 최소 50% 여유 리소스 확보

* **교훈 2: 모니터링 설정**
    - CloudWatch Alarm으로 CPU 80% 이상 시 알림
    - 메모리 사용량 주기적 확인
    - 빌드 실패 시 즉시 대응

* **교훈 3: 빌드 최적화**
    - 불필요한 의존성 제거
    - 캐시 활용으로 빌드 시간 단축
    - 순차 빌드 대신 병렬 빌드 고려

---

## 트러블슈팅 2: AI Server 배포 실패 - progress deadline exceeded

**작성자**: 신동준

---

### 1. 문제 현상 (Problem)

> AI Server Pod이 Ready 상태가 되지 않고 배포 타임아웃 발생

* **문제 1**: `kubectl rollout status` 명령어가 10분 후 타임아웃
* **문제 2**: 새 Pod이 생성되지만 이전 Pod이 계속 실행 중
* **문제 3**: 에러 메시지: "deployment exceeded its progress deadline"

<br>

**[에러 로그]**

```bash
$ kubectl rollout status deployment/signbell-ai-deployment
Waiting for deployment "signbell-ai-deployment" rollout to finish: 1 old replicas are pending termination...
error: deployment "signbell-ai-deployment" exceeded its progress deadline
```

---

### 2. 원인 분석 (Analysis)

> AI Server 초기화 시간이 10분 이상 소요되어 Kubernetes 기본 타임아웃 초과

* **원인 1: 긴 초기화 시간**
    - AI 모델 로딩에 5-7분 소요
    - MediaPipe 초기화에 2-3분 소요
    - Janus WebRTC 연결 설정에 1-2분 소요

* **원인 2: Readiness Probe 설정 부족**
    - `initialDelaySeconds`가 30초로 설정되어 너무 짧음
    - `failureThreshold`가 3으로 설정되어 1분 30초 후 실패 판정
    - 실제 초기화 시간과 설정값 불일치

---

### 3. 해결 방안 (Solution)

> progressDeadlineSeconds와 Readiness Probe 설정을 초기화 시간에 맞게 조정

* **해결 방안 1: progressDeadlineSeconds 증가**
    - `k8s/ai-deployment.yaml` 수정
  ```yaml
  spec:
    progressDeadlineSeconds: 1800  # 30분
  ```

* **해결 방안 2: Readiness Probe 조정**
  ```yaml
  readinessProbe:
    httpGet:
      path: /
      port: 8000
    initialDelaySeconds: 120  # 2분 대기
    periodSeconds: 10
    failureThreshold: 60  # 10분 허용
  ```

* **해결 방안 3: 비동기 배포 전략**
    - Jenkins 파이프라인에서 AI Server 배포를 백그라운드로 실행
    - 다른 서비스 배포와 독립적으로 진행
    - 배포 완료 여부는 별도 모니터링

---

### 4. 교훈 (Lessons Learned)

> 서비스별 특성에 맞는 배포 설정이 필요하며 초기화 시간을 충분히 고려해야 함

* **교훈 1: 서비스별 배포 전략 수립**
    - 빠른 서비스(Frontend, Backend)와 느린 서비스(AI Server) 구분
    - 각 서비스의 초기화 시간 측정 및 문서화
    - 타임아웃 설정을 실제 시간의 1.5배로 여유있게 설정

* **교훈 2: Health Check 최적화**
    - Liveness Probe와 Readiness Probe 분리
    - 초기화 단계별 체크포인트 설정
    - 실패 시 명확한 에러 메시지 제공

* **교훈 3: 배포 모니터링 강화**
    - 배포 진행 상황 실시간 확인
    - 로그 수집 및 분석 자동화
    - 배포 실패 시 즉시 알림

---

## 트러블슈팅 3: Docker 이미지 빌드 실패 - 메모리 부족

**작성자**: 신동준

---

### 1. 문제 현상 (Problem)

> Docker 이미지 빌드 중 메모리 부족으로 프로세스 종료

* **문제 1**: Backend Gradle 빌드 중 "Out of memory" 에러
* **문제 2**: Docker 빌드 프로세스가 중간에 종료됨
* **문제 3**: Jenkins 빌드 로그에 "Killed" 메시지 출력

<br>

**[에러 로그]**

```bash
Step 5/10 : RUN ./gradlew clean build -x test
 ---> Running in abc123def456
> Task :compileJava
Killed
ERROR: failed to solve: process "/bin/sh -c ./gradlew clean build -x test" did not complete successfully
```

---

### 2. 원인 분석 (Analysis)

> Docker 빌드 시 메모리 제한이 없어 호스트 메모리를 과도하게 사용

* **원인 1: Docker 빌드 메모리 무제한**
    - Docker 빌드 시 기본적으로 메모리 제한 없음
    - Gradle 빌드가 1GB 이상 메모리 사용
    - 호스트 메모리 부족으로 OOM Killer 작동

* **원인 2: 멀티 스테이지 빌드 미사용**
    - 빌드 의존성과 런타임 의존성이 혼재
    - 불필요한 파일이 이미지에 포함
    - 이미지 크기 증가로 빌드 시간 증가

---

### 3. 해결 방안 (Solution)

> Docker 빌드 시 메모리 제한을 설정하고 멀티 스테이지 빌드 적용

* **해결 방안 1: Docker 빌드 메모리 제한**
    - Jenkinsfile에서 메모리 제한 추가
  ```groovy
  sh "docker build --memory='1g' --memory-swap='1.5g' -t ${imageFullName} ${dockerfilePath}"
  ```

* **해결 방안 2: Gradle 메모리 설정**
    - `backend/gradle.properties` 파일 수정
  ```properties
  org.gradle.jvmargs=-Xmx512m
  org.gradle.daemon=false
  ```

* **해결 방안 3: 멀티 스테이지 빌드**
    - `backend/Dockerfile` 최적화
  ```dockerfile
  # Build stage
  FROM gradle:8-jdk17 AS builder
  WORKDIR /app
  COPY . .
  RUN gradle clean build -x test
  
  # Runtime stage
  FROM openjdk:17-slim
  WORKDIR /app
  COPY --from=builder /app/build/libs/*.jar app.jar
  ENTRYPOINT ["java", "-jar", "app.jar"]
  ```

---

### 4. 교훈 (Lessons Learned)

> Docker 빌드 최적화와 리소스 제한 설정이 안정적인 배포의 핵심

* **교훈 1: 리소스 제한 필수**
    - 모든 빌드 프로세스에 메모리 제한 설정
    - CPU 제한도 고려하여 다른 프로세스 영향 최소화
    - 제한값은 실제 사용량의 1.2배로 설정

* **교훈 2: 이미지 최적화**
    - 멀티 스테이지 빌드로 이미지 크기 50% 감소
    - .dockerignore 파일로 불필요한 파일 제외
    - 레이어 캐싱 활용으로 빌드 시간 단축

* **교훈 3: 빌드 환경 표준화**
    - 로컬과 CI/CD 환경의 Docker 설정 통일
    - 빌드 스크립트 문서화
    - 팀원 간 빌드 환경 공유

---

## 트러블슈팅 4: CORS 중복 헤더 문제

**작성자**: 신동준

---

### 1. 문제 현상 (Problem)

> 브라우저에서 API 요청 시 CORS 에러 발생, 서버는 200 OK 응답

* **문제 1**: 브라우저 콘솔에 CORS policy 에러 메시지
* **문제 2**: 네트워크 탭에서 `POST 200 (OK)` 이지만 `net::ERR_FAILED`
* **문제 3**: Access-Control-Allow-Origin 헤더에 여러 값 포함

<br>

**[에러 로그]**

```
Access to fetch at 'https://api.signbell.app/api/proxy/janus' 
from origin 'https://www.signbell.app' has been blocked by CORS policy: 
The 'Access-Control-Allow-Origin' header contains multiple values 
'https://www.signbell.app, *', but only one is allowed.

POST https://api.signbell.app/api/proxy/janus net::ERR_FAILED 200 (OK)
```

---

### 2. 원인 분석 (Analysis)

> Janus 서버와 Spring Boot가 동시에 CORS 헤더를 추가하여 중복 발생

* **원인 1: 외부 서버의 CORS 헤더**
    - Janus 서버가 자체적으로 `Access-Control-Allow-Origin: *` 헤더 전송
    - 프록시를 통해 응답을 전달할 때 헤더가 그대로 유지됨

* **원인 2: Spring Boot의 CORS 설정**
    - SecurityConfig에서 `Access-Control-Allow-Origin: https://www.signbell.app` 추가
    - 두 헤더가 합쳐져서 `https://www.signbell.app, *` 형태로 전송

* **원인 3: 브라우저의 CORS 정책**
    - Access-Control-Allow-Origin 헤더는 단일 값만 허용
    - 여러 값이 있으면 브라우저가 응답을 차단
    - 서버는 정상 응답했지만 브라우저가 거부

---

### 3. 해결 방안 (Solution)

> 프록시 컨트롤러에서 외부 서버의 CORS 헤더를 제거하고 Spring 설정만 사용

* **해결 방안 1: CORS 헤더 필터링 메서드 추가**
  ```java
  private HttpHeaders removeCorsHeaders(HttpHeaders originalHeaders) {
      HttpHeaders cleanHeaders = new HttpHeaders();
      originalHeaders.forEach((key, value) -> {
          if (!key.toLowerCase().startsWith("access-control-")) {
              cleanHeaders.addAll(key, value);
          }
      });
      return cleanHeaders;
  }
  ```

* **해결 방안 2: 프록시 응답에 적용**
  ```java
  @PostMapping(value = "/**")
  public ResponseEntity<String> proxyJanusPost(@RequestBody String body) {
      ResponseEntity<String> response = restTemplate.postForEntity(janusUrl, entity, String.class);
      
      return ResponseEntity
          .status(response.getStatusCode())
          .headers(removeCorsHeaders(response.getHeaders()))
          .body(response.getBody());
  }
  ```

* **해결 방안 3: SecurityConfig CORS 설정 최적화**
  ```java
  @Bean
  public CorsConfigurationSource corsConfigurationSource() {
      CorsConfiguration configuration = new CorsConfiguration();
      configuration.setAllowedOrigins(Arrays.asList(
          "https://www.signbell.app",
          "https://api.signbell.app"
      ));
      configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "OPTIONS"));
      configuration.setAllowedHeaders(Arrays.asList("*"));
      configuration.setAllowCredentials(true);
      configuration.setMaxAge(3600L);
      
      UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
      source.registerCorsConfiguration("/**", configuration);
      return source;
  }
  ```

---

### 4. 교훈 (Lessons Learned)

> CORS 설정은 한 곳에서만 관리하고 외부 서버 응답은 필터링해야 함

* **교훈 1: CORS 설정 중앙화**
    - 컨트롤러 레벨 `@CrossOrigin` 어노테이션 사용 지양
    - SecurityConfig에서 전역 CORS 설정 관리
    - 프록시 사용 시 외부 헤더 제거 필수

* **교훈 2: 브라우저 개발자 도구 활용**
    - Network 탭에서 응답 헤더 확인
    - Console 탭에서 CORS 에러 메시지 분석
    - 200 OK인데 실패하는 경우 CORS 의심

* **교훈 3: 프록시 패턴 이해**
    - 프록시는 요청/응답을 중계하는 역할
    - 외부 서버의 헤더를 그대로 전달하면 충돌 가능
    - 필요한 헤더만 선택적으로 전달

---

## 트러블슈팅 5: Kubernetes Pod Pending 상태 지속

**작성자**: 신동준

---

### 1. 문제 현상 (Problem)

> 새로운 Pod이 생성되지만 Pending 상태에서 진행되지 않음

* **문제 1**: `kubectl get pods` 실행 시 Pod 상태가 Pending
* **문제 2**: 배포가 완료되지 않고 타임아웃 발생
* **문제 3**: 기존 Pod은 정상 실행 중

<br>

**[Pod 상태]**

```bash
$ kubectl get pods
NAME                                      READY   STATUS    RESTARTS   AGE
signbell-backend-deployment-abc123-xyz   1/1     Running   0          10m
signbell-backend-deployment-def456-uvw   0/1     Pending   0          5m
```

---

### 2. 원인 분석 (Analysis)

> 클러스터 리소스 부족으로 새 Pod을 스케줄링할 수 없음

* **원인 1: 노드 리소스 부족**
    - 모든 노드의 CPU/메모리가 거의 사용 중
    - 새 Pod의 리소스 요청량을 만족하는 노드 없음
    - `kubectl describe pod`에서 "Insufficient cpu" 메시지 확인

* **원인 2: PersistentVolumeClaim 문제**
    - PVC가 Pending 상태로 볼륨 바인딩 실패
    - 스토리지 클래스 설정 오류
    - 가용 볼륨 부족

---

### 3. 해결 방안 (Solution)

> 노드 리소스 확인 후 스케일 업 또는 리소스 요청량 조정

* **해결 방안 1: 노드 리소스 확인**
  ```bash
  # 노드 리소스 사용량 확인
  kubectl top nodes
  
  # Pod 이벤트 확인
  kubectl describe pod <pod-name>
  
  # 리소스 부족 확인
  kubectl get events --sort-by='.lastTimestamp'
  ```

* **해결 방안 2: 노드 추가 또는 스케일 업**
  ```bash
  # EKS 노드 그룹 스케일 업
  eksctl scale nodegroup --cluster=signbell-cluster \
    --name=ng-signbell --nodes=3 --nodes-min=2 --nodes-max=4
  ```

* **해결 방안 3: Pod 리소스 요청량 조정**
  ```yaml
  resources:
    requests:
      cpu: "250m"      # 500m → 250m 감소
      memory: "256Mi"  # 512Mi → 256Mi 감소
    limits:
      cpu: "1"
      memory: "1Gi"
  ```

---

### 4. 교훈 (Lessons Learned)

> 클러스터 리소스를 주기적으로 모니터링하고 여유 용량을 확보해야 함

* **교훈 1: 리소스 모니터링**
    - 노드 CPU/메모리 사용률 80% 이하 유지
    - 자동 스케일링 설정으로 리소스 부족 방지
    - CloudWatch 알람으로 리소스 부족 사전 감지

* **교훈 2: 리소스 요청량 최적화**
    - 실제 사용량 측정 후 요청량 설정
    - 과도한 요청량은 스케줄링 실패 원인
    - requests는 최소값, limits는 최대값으로 설정

* **교훈 3: 배포 전 리소스 확인**
    - 배포 전 클러스터 리소스 상태 확인
    - 여유 리소스 부족 시 노드 추가 후 배포
    - 배포 실패 시 즉시 롤백

---

**작성일**: 2025-11-02  
**프로젝트**: SignBell
