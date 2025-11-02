# UI/UX 트러블슈팅 모음

> **작성일**: 2025-11-02  
> **프로젝트**: SignBell

---

## 트러블슈팅 1: 웹캠 권한 거부 후 재요청 불가

**작성자**: 신동준

---

### 1. 문제 현상 (Problem)

> 사용자가 웹캠 권한을 거부한 후 다시 허용하려 해도 웹캠이 활성화되지 않음

* **문제 1**: 웹캠 권한 거부 후 "웹캠 켜기" 버튼 클릭 시 반응 없음
* **문제 2**: 브라우저 설정에서 권한을 허용해도 페이지에서 인식하지 못함
* **문제 3**: 페이지 새로고침 후에도 웹캠이 활성화되지 않음

<br>

**[콘솔 에러]**

```javascript
DOMException: Permission denied
NotAllowedError: Permission denied by system
```

---

### 2. 원인 분석 (Analysis)

> 브라우저가 권한 거부를 기억하고 있어 재요청이 차단됨

* **원인 1: 브라우저 권한 캐싱**
    - 사용자가 한 번 거부하면 브라우저가 해당 도메인의 권한 요청을 차단
    - `navigator.mediaDevices.getUserMedia()` 호출 시 즉시 거부
    - 페이지 새로고침으로는 해결되지 않음

* **원인 2: 에러 처리 부족**
    - 권한 거부 시 사용자에게 안내 메시지 미제공
    - 권한 설정 방법을 알려주지 않음
    - 재시도 로직 없음

---

### 3. 해결 방안 (Solution)

> 권한 상태를 확인하고 거부 시 명확한 안내 메시지 제공

* **해결 방안 1: 권한 상태 확인**
  ```javascript
  const checkCameraPermission = async () => {
    try {
      const result = await navigator.permissions.query({ name: 'camera' });
      
      if (result.state === 'denied') {
        setPermissionDenied(true);
        setErrorMessage('웹캠 권한이 거부되었습니다. 브라우저 설정에서 권한을 허용해주세요.');
      } else if (result.state === 'granted') {
        await startWebcam();
      } else {
        // prompt 상태: 권한 요청
        await requestWebcamPermission();
      }
    } catch (error) {
      console.error('권한 확인 실패:', error);
    }
  };
  ```

* **해결 방안 2: 사용자 안내 모달**
  ```jsx
  {permissionDenied && (
    <Modal>
      <h3>웹캠 권한이 필요합니다</h3>
      <p>브라우저 주소창 왼쪽의 자물쇠 아이콘을 클릭하여 카메라 권한을 허용해주세요.</p>
      <ol>
        <li>주소창 왼쪽 자물쇠 아이콘 클릭</li>
        <li>"카메라" 항목 찾기</li>
        <li>"허용"으로 변경</li>
        <li>페이지 새로고침</li>
      </ol>
      <button onClick={() => window.location.reload()}>새로고침</button>
    </Modal>
  )}
  ```

* **해결 방안 3: 에러 처리 개선**
  ```javascript
  const requestWebcamPermission = async () => {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ video: true });
      setWebcamStream(stream);
      setWebcamEnabled(true);
    } catch (error) {
      if (error.name === 'NotAllowedError') {
        setPermissionDenied(true);
        setErrorMessage('웹캠 권한이 거부되었습니다.');
      } else if (error.name === 'NotFoundError') {
        setErrorMessage('웹캠을 찾을 수 없습니다.');
      } else {
        setErrorMessage('웹캠 활성화에 실패했습니다.');
      }
    }
  };
  ```

---

### 4. 교훈 (Lessons Learned)

> 브라우저 권한 요청은 한 번의 기회이므로 명확한 안내와 에러 처리가 필수

* **교훈 1: 권한 요청 전 안내**
    - 권한 요청 전에 왜 필요한지 설명
    - 사용자가 이해하고 허용할 수 있도록 유도
    - 거부 시 재요청이 어렵다는 점 고려

* **교훈 2: 에러 메시지 명확화**
    - 에러 타입별로 다른 메시지 제공
    - 해결 방법을 단계별로 안내
    - 시각적 가이드 (스크린샷) 제공

* **교훈 3: 대체 방안 제공**
    - 웹캠 없이도 일부 기능 사용 가능하도록
    - 모바일 환경 고려
    - 권한 거부 시 다른 입력 방법 제안

---

## 트러블슈팅 2: 모바일에서 사이드바 스크롤 불가

**작성자**: 신동준

---

### 1. 문제 현상 (Problem)

> 모바일 환경에서 사이드바 내부 콘텐츠를 스크롤할 수 없음

* **문제 1**: 사이드바가 열리면 배경 페이지가 스크롤됨
* **문제 2**: 사이드바 내부 콘텐츠가 길어도 스크롤되지 않음
* **문제 3**: 사이드바 외부를 터치하면 사이드바가 닫힘

<br>

**[재현 방법]**

1. 모바일 브라우저에서 메인 페이지 접속
2. "실시간 퀴즈" 버튼 클릭하여 사이드바 열기
3. 사이드바 내부를 위아래로 스크롤 시도
4. 배경 페이지가 스크롤되고 사이드바는 고정됨

---

### 2. 원인 분석 (Analysis)

> 모바일 브라우저의 터치 이벤트가 사이드바를 관통하여 배경 페이지로 전달됨

* **원인 1: 터치 이벤트 버블링**
    - 사이드바 내부 터치 이벤트가 배경으로 전파
    - `overflow: auto`만으로는 터치 스크롤 제어 불가
    - iOS Safari에서 특히 심함

* **원인 2: body 스크롤 미차단**
    - 사이드바가 열려도 body 스크롤이 활성화 상태
    - `position: fixed` 사이드바가 body 스크롤에 영향받음
    - 모바일에서 스크롤 동작이 예측 불가능

---

### 3. 해결 방안 (Solution)

> 사이드바 열릴 때 body 스크롤을 차단하고 터치 이벤트 전파 방지

* **해결 방안 1: body 스크롤 차단**
  ```javascript
  useEffect(() => {
    if (sidebarOpen) {
      // 사이드바 열릴 때 body 스크롤 차단
      document.body.style.overflow = 'hidden';
      document.body.style.position = 'fixed';
      document.body.style.width = '100%';
    } else {
      // 사이드바 닫힐 때 복원
      document.body.style.overflow = '';
      document.body.style.position = '';
      document.body.style.width = '';
    }
    
    return () => {
      document.body.style.overflow = '';
      document.body.style.position = '';
      document.body.style.width = '';
    };
  }, [sidebarOpen]);
  ```

* **해결 방안 2: 터치 이벤트 전파 방지**
  ```javascript
  const handleTouchMove = (e) => {
    // 사이드바 내부 스크롤 가능 영역인지 확인
    const target = e.target;
    const scrollableElement = target.closest('.sidebar-content');
    
    if (scrollableElement) {
      // 스크롤 가능 영역이면 이벤트 전파 허용
      return;
    }
    
    // 그 외 영역은 이벤트 전파 차단
    e.preventDefault();
  };
  
  <div 
    className={styles.sidebar}
    onTouchMove={handleTouchMove}
  >
    <div className={styles.sidebarContent}>
      {/* 스크롤 가능한 콘텐츠 */}
    </div>
  </div>
  ```

* **해결 방안 3: CSS 스크롤 최적화**
  ```scss
  .sidebar {
    position: fixed;
    right: 0;
    top: 0;
    height: 100vh;
    overflow: hidden;
    
    .sidebarContent {
      height: 100%;
      overflow-y: auto;
      -webkit-overflow-scrolling: touch; // iOS 부드러운 스크롤
      overscroll-behavior: contain; // 스크롤 끝에서 배경 스크롤 방지
    }
  }
  ```

---

### 4. 교훈 (Lessons Learned)

> 모바일 환경에서는 터치 이벤트와 스크롤 동작을 명시적으로 제어해야 함

* **교훈 1: 모바일 우선 테스트**
    - 개발 초기부터 모바일 환경 테스트
    - 실제 디바이스에서 확인 필수
    - Chrome DevTools 모바일 에뮬레이터는 한계 있음

* **교훈 2: 스크롤 동작 명확화**
    - 어떤 요소가 스크롤 가능한지 명확히 정의
    - 중첩된 스크롤 영역 최소화
    - `overscroll-behavior` 속성 활용

* **교훈 3: iOS Safari 특수 처리**
    - `-webkit-overflow-scrolling: touch` 필수
    - `position: fixed` 사용 시 주의
    - 100vh 대신 `window.innerHeight` 사용 고려

---

## 트러블슈팅 3: 모달 뒤 배경 스크롤 문제

**작성자**: 신동준

---

### 1. 문제 현상 (Problem)

> 모달이 열려있을 때 배경 페이지가 스크롤되어 UX 저해

* **문제 1**: 모달 내부를 스크롤하려 하면 배경 페이지가 스크롤됨
* **문제 2**: 모달 오버레이를 터치하면 배경이 움직임
* **문제 3**: 모달 닫은 후 스크롤 위치가 변경됨

<br>

**[재현 방법]**

1. 방 만들기 버튼 클릭하여 모달 열기
2. 모달 외부 영역을 스크롤 시도
3. 배경 페이지가 스크롤되는 것 확인

---

### 2. 원인 분석 (Analysis)

> 모달이 열려도 body 스크롤이 활성화되어 있어 터치 이벤트가 배경으로 전달됨

* **원인 1: body 스크롤 미차단**
    - 모달 컴포넌트에서 body 스크롤 제어 없음
    - `position: fixed` 모달이 body 스크롤에 영향받음

* **원인 2: 스크롤 위치 저장 미흡**
    - 모달 열릴 때 현재 스크롤 위치 저장 안 함
    - 모달 닫힐 때 원래 위치로 복원 안 됨

---

### 3. 해결 방안 (Solution)

> 모달 열릴 때 스크롤 위치를 저장하고 body 스크롤 차단

* **해결 방안 1: 스크롤 위치 저장 및 복원**
  ```javascript
  const [scrollPosition, setScrollPosition] = useState(0);
  
  useEffect(() => {
    if (modalOpen) {
      // 현재 스크롤 위치 저장
      const currentScroll = window.pageYOffset;
      setScrollPosition(currentScroll);
      
      // body 스크롤 차단
      document.body.style.overflow = 'hidden';
      document.body.style.position = 'fixed';
      document.body.style.top = `-${currentScroll}px`;
      document.body.style.width = '100%';
    } else {
      // body 스크롤 복원
      document.body.style.overflow = '';
      document.body.style.position = '';
      document.body.style.top = '';
      document.body.style.width = '';
      
      // 스크롤 위치 복원
      window.scrollTo(0, scrollPosition);
    }
  }, [modalOpen]);
  ```

* **해결 방안 2: 모달 컴포넌트 개선**
  ```jsx
  const Modal = ({ isOpen, onClose, children }) => {
    useEffect(() => {
      if (isOpen) {
        const scrollY = window.scrollY;
        document.body.style.position = 'fixed';
        document.body.style.top = `-${scrollY}px`;
        document.body.style.width = '100%';
        
        return () => {
          document.body.style.position = '';
          document.body.style.top = '';
          document.body.style.width = '';
          window.scrollTo(0, scrollY);
        };
      }
    }, [isOpen]);
    
    if (!isOpen) return null;
    
    return (
      <div className={styles.modalOverlay} onClick={onClose}>
        <div className={styles.modalContent} onClick={(e) => e.stopPropagation()}>
          {children}
        </div>
      </div>
    );
  };
  ```

* **해결 방안 3: CSS 최적화**
  ```scss
  .modalOverlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.5);
    z-index: 1000;
    overflow-y: auto;
    
    .modalContent {
      position: relative;
      margin: 50px auto;
      max-width: 500px;
      background: white;
      border-radius: 16px;
      padding: 32px;
    }
  }
  ```

---

### 4. 교훈 (Lessons Learned)

> 모달 같은 오버레이 UI는 배경 스크롤 제어가 필수

* **교훈 1: 스크롤 제어 표준화**
    - 모든 모달/사이드바에 동일한 스크롤 제어 로직 적용
    - 커스텀 훅으로 재사용 가능하게 구현
    - `useScrollLock` 훅 생성

* **교훈 2: 스크롤 위치 복원**
    - 모달 닫을 때 원래 스크롤 위치로 복원
    - 사용자 경험 향상
    - 특히 긴 페이지에서 중요

* **교훈 3: 이벤트 전파 차단**
    - 모달 내부 클릭 시 `stopPropagation()` 사용
    - 오버레이 클릭 시에만 모달 닫기
    - 의도하지 않은 닫힘 방지

---

## 트러블슈팅 4: CSS 애니메이션 끊김 현상

**작성자**: 신동준

---

### 1. 문제 현상 (Problem)

> 페이지 전환 및 모달 애니메이션이 부드럽지 않고 끊김

* **문제 1**: 사이드바 슬라이드 인 애니메이션이 버벅임
* **문제 2**: 모달 스케일 인 애니메이션이 부자연스러움
* **문제 3**: 모바일에서 특히 심함

<br>

**[재현 방법]**

1. 메인 페이지에서 실시간 퀴즈 버튼 클릭
2. 사이드바가 열리는 애니메이션 관찰
3. 끊김 현상 확인

---

### 2. 원인 분석 (Analysis)

> CSS 애니메이션에서 reflow를 유발하는 속성을 사용하여 성능 저하

* **원인 1: reflow 유발 속성 사용**
    - `left`, `right`, `width`, `height` 같은 레이아웃 속성 애니메이션
    - 브라우저가 매 프레임마다 레이아웃 재계산
    - GPU 가속 미사용

* **원인 2: will-change 미사용**
    - 브라우저에게 애니메이션 힌트 제공 안 함
    - 최적화 기회 상실

---

### 3. 해결 방안 (Solution)

> transform과 opacity만 사용하고 GPU 가속 활성화

* **해결 방안 1: transform 사용**
  ```scss
  // 기존 (reflow 유발)
  @keyframes slideInRight {
    from {
      right: -400px;
    }
    to {
      right: 0;
    }
  }
  
  // 개선 (GPU 가속)
  @keyframes slideInRight {
    from {
      transform: translateX(100%);
    }
    to {
      transform: translateX(0);
    }
  }
  
  .sidebar {
    position: fixed;
    right: 0;
    animation: slideInRight 0.3s ease-out;
  }
  ```

* **해결 방안 2: will-change 추가**
  ```scss
  .sidebar {
    will-change: transform;
    
    &.closing {
      will-change: auto; // 애니메이션 끝나면 제거
    }
  }
  
  .modal {
    will-change: transform, opacity;
  }
  ```

* **해결 방안 3: 하드웨어 가속 강제**
  ```scss
  .animated-element {
    transform: translateZ(0); // GPU 레이어 생성
    backface-visibility: hidden; // 뒷면 렌더링 방지
  }
  ```

---

### 4. 교훈 (Lessons Learned)

> CSS 애니메이션은 transform과 opacity만 사용하고 GPU 가속을 활용해야 함

* **교훈 1: 애니메이션 가능 속성 제한**
    - transform, opacity만 애니메이션
    - left, right, width, height 사용 금지
    - 성능 테스트 필수

* **교훈 2: 브라우저 최적화 활용**
    - will-change로 브라우저에 힌트 제공
    - 애니메이션 끝나면 will-change 제거
    - 과도한 사용은 오히려 성능 저하

* **교훈 3: 모바일 성능 우선**
    - 모바일에서 애니메이션 테스트
    - 60fps 유지 목표
    - 필요시 애니메이션 단순화

---

## 트러블슈팅 5: 반응형 디자인 브레이크포인트 문제

**작성자**: 신동준

---

### 1. 문제 현상 (Problem)

> 특정 화면 크기에서 레이아웃이 깨지거나 요소가 겹침

* **문제 1**: 768px 근처에서 버튼이 두 줄로 나뉘어짐
* **문제 2**: 태블릿 가로 모드에서 사이드바가 화면을 가림
* **문제 3**: 작은 모바일 기기에서 텍스트가 잘림

<br>

**[재현 방법]**

1. 브라우저 창 크기를 768px로 조정
2. 메인 페이지의 기능 버튼 영역 확인
3. 버튼이 두 줄로 나뉘어지는 것 확인

---

### 2. 원인 분석 (Analysis)

> 고정된 브레이크포인트만 사용하여 중간 크기 화면 대응 부족

* **원인 1: 브레이크포인트 부족**
    - 데스크톱(1200px)과 모바일(768px)만 고려
    - 태블릿 가로(1024px), 작은 모바일(375px) 미고려
    - 중간 크기에서 레이아웃 불안정

* **원인 2: 고정 크기 사용**
    - px 단위로 고정된 크기 사용
    - 화면 크기에 따른 유연성 부족
    - min-width, max-width 미사용

---

### 3. 해결 방안 (Solution)

> 더 세분화된 브레이크포인트와 유연한 단위 사용

* **해결 방안 1: 브레이크포인트 추가**
  ```scss
  // 기존
  @media (max-width: 1199px) { /* 태블릿 */ }
  @media (max-width: 767px) { /* 모바일 */ }
  
  // 개선
  @media (max-width: 1399px) { /* 작은 데스크톱 */ }
  @media (max-width: 1199px) { /* 태블릿 가로 */ }
  @media (max-width: 991px) { /* 태블릿 세로 */ }
  @media (max-width: 767px) { /* 모바일 가로 */ }
  @media (max-width: 575px) { /* 모바일 세로 */ }
  ```

* **해결 방안 2: 유연한 단위 사용**
  ```scss
  .button {
    // 고정 크기 대신 유연한 크기
    width: clamp(200px, 30vw, 400px);
    padding: clamp(12px, 2vw, 24px);
    font-size: clamp(14px, 1.5vw, 18px);
  }
  
  .container {
    max-width: min(1200px, 90vw);
    padding: clamp(16px, 5vw, 40px);
  }
  ```

* **해결 방안 3: 컨테이너 쿼리 활용**
  ```scss
  .card-container {
    container-type: inline-size;
  }
  
  .card {
    display: flex;
    flex-direction: column;
    
    @container (min-width: 600px) {
      flex-direction: row;
    }
  }
  ```

---

### 4. 교훈 (Lessons Learned)

> 반응형 디자인은 다양한 화면 크기를 고려하고 유연한 단위를 사용해야 함

* **교훈 1: 실제 디바이스 테스트**
    - 다양한 화면 크기에서 테스트
    - 실제 디바이스 사용 권장
    - 브레이크포인트 사이 크기도 확인

* **교훈 2: 유연한 레이아웃**
    - Flexbox, Grid 활용
    - 고정 크기 최소화
    - clamp(), min(), max() 함수 활용

* **교훈 3: 모바일 우선 설계**
    - 작은 화면부터 디자인
    - 점진적으로 큰 화면 대응
    - 불필요한 요소 제거

---

**작성일**: 2025-11-02  
**프로젝트**: SignBell
