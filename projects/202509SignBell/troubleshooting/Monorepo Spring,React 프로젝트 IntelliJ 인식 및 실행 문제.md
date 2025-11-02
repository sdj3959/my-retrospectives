## 트러블슈팅: Monorepo Spring/React 프로젝트 IntelliJ 인식 및 실행 문제

### 1. 문제 현상

Git 저장소에서 Monorepo (Spring Boot `backend`, React `frontend`) 프로젝트를 클론 또는 Pull 받은 후, IntelliJ IDEA에서 다음과 같은 문제가 발생했습니다.

*   `backend` (Spring Boot) 모듈의 `BackendApplication.java` 파일에서 실행(Run) 버튼이 비활성화됨.
*   IntelliJ IDEA가 프로젝트 구조를 올바르게 인식하지 못하여 모듈이 제대로 로드되지 않거나, 수동으로 모듈을 추가해야 하는 상황 발생.
*   `.idea/` 디렉토리가 `.gitignore`에 추가되었음에도 불구하고 Git 저장소에 커밋되어 팀원 간 IntelliJ 설정 충돌 발생.

### 2. 원인 분석

1.  **`.idea/` 디렉토리의 Git 추적:**
    *   `.gitignore` 파일은 Git이 **추적하지 않는(Untracked)** 파일에 대해서만 작동합니다. `.idea/` 디렉토리가 이미 Git 저장소에 커밋되어 추적되고 있었기 때문에, `.gitignore`에 추가하더라도 Git이 이를 무시하지 않고 계속 관리했습니다. 이로 인해 각 개발자의 로컬 IntelliJ 설정이 저장소에 공유되어 충돌을 야기했습니다.
2.  **`settings.gradle` 파일의 불완전한 설정:**
    *   Monorepo의 루트 디렉토리에 있는 `settings.gradle` 파일이 `rootProject.name = 'backend'`와 같이 단일 모듈만 포함하도록 설정되어 있었습니다. 이로 인해 Gradle 및 IntelliJ는 `SignBell-App` 전체를 Monorepo로 인식하지 못하고, `backend` 모듈만을 프로젝트의 루트로 잘못 인식했습니다.
3.  **IntelliJ IDEA의 프로젝트 동기화 문제:**
    *   위 두 가지 원인으로 인해 IntelliJ IDEA가 Gradle 프로젝트의 전체 구조를 올바르게 파악하지 못했고, 결과적으로 `backend` 모듈을 Spring Boot 애플리케이션으로 인식하지 못하여 실행 구성을 생성하거나 활성화하지 못했습니다.

### 3. 해결 방안

다음 단계를 순서대로 수행하여 문제를 해결했습니다.

#### 3.1. `.idea/` 디렉토리 Git 추적 제외

`.idea/` 디렉토리는 각 개발자의 로컬 IntelliJ 설정이므로 Git 저장소에서 완전히 제거하고 `.gitignore`에 추가하여 추적을 방지해야 합니다.

1.  **`.gitignore` 파일 확인:**
    *   프로젝트 루트 디렉토리의 `.gitignore` 파일에 `.idea/`가 포함되어 있는지 확인합니다. (없다면 추가합니다.)
        ```gitignore
        # .gitignore
        .idea/
        *.iml
        .gradle/
        build/
        ```
2.  **Git 추적 대상에서 `.idea/` 제거:**
    *   터미널을 열고 프로젝트 루트 디렉토리로 이동합니다.
    *   다음 명령어를 실행하여 `.idea/` 디렉토리를 Git의 추적 대상에서 완전히 제거합니다. (로컬 파일은 삭제되지 않습니다.)
        ```bash
        git rm -r --cached .idea
        ```
3.  **변경 사항 커밋 및 푸시:**
    *   변경 사항을 커밋하고 원격 저장소에 푸시합니다.
        ```bash
        git commit -m "Remove .idea/ from git tracking"
        git push origin main # 또는 master 등 브랜치명
        ```
    *   **팀원들에게 알림:** 이 변경 사항을 Pull 받은 팀원들은 각자 로컬 프로젝트 디렉토리에서 기존의 `.idea/` 폴더를 수동으로 삭제한 후, IntelliJ를 재시작하여 `.idea/` 폴더가 새로 생성되도록 해야 합니다.

#### 3.2. `settings.gradle` 파일 수정

Monorepo의 모든 서브 모듈을 Gradle이 올바르게 인식하도록 `settings.gradle` 파일을 수정합니다.

1.  **`settings.gradle` 파일 수정:**
    *   프로젝트 루트 디렉토리의 `settings.gradle` 파일을 열고 다음과 같이 수정합니다.
        ```gradle
        // settings.gradle
        rootProject.name = 'SignBell-App' // Monorepo 전체의 이름

        // 서브 모듈들을 포함시킵니다.
        include 'backend'
        include 'frontend'
        ```
    *   `frontend`가 Gradle 모듈이 아니더라도 IntelliJ가 프로젝트 구조를 파악하는 데 도움이 되므로 `include 'frontend'`를 유지하는 것이 좋습니다.
2.  **변경 사항 커밋 및 푸시:**
    *   수정된 `settings.gradle` 파일을 커밋하고 원격 저장소에 푸시합니다.
        ```bash
        git commit -m "Update settings.gradle for monorepo module inclusion"
        git push origin main
        ```

#### 3.3. IntelliJ IDEA 프로젝트 재로드 및 동기화

`settings.gradle` 파일 수정 및 `.idea/` 제거 후, IntelliJ IDEA가 프로젝트 구조를 새로 인식하도록 강제합니다.

1.  **IntelliJ IDEA 캐시 무효화 및 재시작 (선택 사항이지만 권장):**
    *   `File` > `Invalidate Caches / Restart...`를 선택하고 'Invalidate and Restart'를 클릭합니다. (이전 `.idea/` 잔여 설정으로 인한 문제를 방지합니다.)
2.  **Gradle 프로젝트 재로드:**
    *   IntelliJ IDEA가 재시작되거나 프로젝트가 열리면, 우측 하단에 Gradle 동기화 메시지가 나타날 수 있습니다.
    *   만약 자동으로 동기화되지 않거나 문제가 발생하면, IntelliJ IDEA의 'Gradle' 탭을 열고 상단의 **새로고침 아이콘 (Reload All Gradle Projects)**을 클릭하여 강제로 프로젝트를 재로드합니다.
3.  **`build.gradle` 파일에서 프로젝트 연결:**
    *   `backend` 모듈의 `build.gradle` 파일을 IntelliJ IDEA에서 엽니다.
    *   파일 내용 중 아무 곳이나 **우클릭**한 후, 컨텍스트 메뉴에서 **`Link Gradle Project`** (또는 `Import Gradle Project`)를 선택합니다.
    *   이 작업은 IntelliJ에게 해당 `build.gradle` 파일을 기반으로 Gradle 프로젝트를 명시적으로 연결하도록 지시합니다.

### 4. 결과 확인

위 단계를 모두 수행한 후, IntelliJ IDEA는 `SignBell-App`을 올바른 Monorepo 구조로 인식하고 `backend` 및 `frontend` 모듈을 정상적으로 로드합니다.

*   `backend` 모듈의 `BackendApplication.java` 파일에서 **실행(Run) 버튼이 활성화**됩니다.
*   IntelliJ IDEA의 'Project' 뷰에서 Monorepo 구조가 올바르게 표시됩니다.
*   팀원들도 클론/Pull 후 위 단계를 (특히 `.idea/` 삭제 및 프로젝트 재로드) 한 번만 수행하면 동일하게 프로젝트를 인식하고 개발을 시작할 수 있습니다.

---
