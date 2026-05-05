# PILL MOA 💊

사용자의 복약 안전성과 편의성을 높이기 위한 **의약 정보 통합 Android 앱** 프로젝트입니다.  
현재 저장소에는 기능 설명과 기본 앱 모듈 스켈레톤이 포함되어 있으며, 아래 문서는 **현 코드 기준 분석 + 통합 설계/아키텍처 제안**을 한 번에 정리한 통합 README 입니다.

---

## 1) 프로젝트 개요

### 목표
- 의약품 탐색, 처방 이력 확인, 복약 알림, 개인 설정을 한 앱에서 제공
- 사용자의 복약 순응도(Adherence) 향상
- 핵심 건강 정보를 빠르게 조회 가능한 UX 제공

### 현재 저장소 상태(코드 분석 결과)
- 루트 `README.md`: 서비스 목표/주요 기능 요약 문서
- `app/A.java`: 앱 모듈 디렉터리 구조 메모(초기 템플릿 성격)

즉, **기능 코드 구현 단계 이전의 기획/스캐폴딩 단계**에 가깝고, 실제 Activity/Fragment/Repository/DB/API 구현은 아직 반영되어 있지 않습니다.

---

## 2) 주요 기능 요구사항(정의)

1. **의약품 정보 검색**
   - 약 이름/성분/키워드 기반 검색
   - 상세 정보: 효능, 용법, 주의사항, 상호작용

2. **처방전 열람**
   - 과거 처방 목록 및 상세 이력 조회
   - 날짜/병원/약품 기준 필터

3. **복약 알림**
   - 하루 복약 스케줄 등록
   - 알림 반복, 미복용 체크, 재알림

4. **환경설정**
   - 알림 시간 정책, 폰트/테마, 개인정보 동의 상태

---

## 3) 권장 아키텍처 (통합 설계)

Android(Java) 프로젝트의 확장성과 유지보수를 위해 **Layered + MVVM** 구조를 권장합니다.

### 3.1 아키텍처 레이어

- **Presentation Layer**
  - Activity/Fragment, ViewModel, UI State
  - 사용자 이벤트 처리, 상태 렌더링

- **Domain Layer**
  - UseCase (예: `SearchMedicineUseCase`, `GetPrescriptionHistoryUseCase`)
  - 비즈니스 규칙 캡슐화

- **Data Layer**
  - Repository 인터페이스/구현
  - Remote DataSource(API), Local DataSource(Room/SharedPreferences)

### 3.2 패키지 구조 예시

```text
app/src/main/java/com/pillmoa/
  presentation/
    home/
    search/
    prescription/
    reminder/
    settings/
  domain/
    model/
    repository/
    usecase/
  data/
    remote/
      api/
      dto/
    local/
      db/
      entity/
      dao/
      pref/
    repository/
  common/
    util/
    base/
    di/
```

---

## 4) 기능별 상세 설계

### 4.1 의약품 검색

**화면 흐름**  
검색 입력 → 결과 리스트 → 상세 화면

**핵심 컴포넌트**
- `MedicineSearchViewModel`
- `SearchMedicineUseCase`
- `MedicineRepository`
- `MedicineApiService`

**캐시 전략(권장)**
- 최근 검색어: Local 저장
- 상세 정보: TTL 캐시 적용(예: 24시간)

### 4.2 처방전 열람

**데이터 모델 예시**
- `Prescription(id, date, hospitalName, doctorName)`
- `PrescriptionItem(prescriptionId, medicineName, dosage, frequency, duration)`

**조회 기능**
- 기간 필터
- 처방 상세 drill-down

### 4.3 복약 알림

**권장 기술**
- `WorkManager`(지연/보장 실행)
- `AlarmManager` + `BroadcastReceiver`(정시성이 중요한 알림)
- 알림 탭 시 복약 체크 화면 이동

**예외 처리 시나리오**
- 기기 재부팅 후 알림 재등록
- 앱 업데이트/시간대 변경 시 스케줄 재계산

### 4.4 환경설정

- 알림 ON/OFF 및 조용한 시간대
- 개인정보 처리방침/약관 링크
- 백업/복원 정책(선택)

---

## 5) 데이터/연동 설계

### 5.1 외부 연동 포인트

1. **의약품 공공/상용 API**
   - 약 정보, 성분, 효능/부작용 데이터
2. **인증/사용자 식별(선택)**
   - 로그인 필요 시 토큰 기반 인증
3. **푸시/알림 인프라(선택)**
   - 로컬 알림 우선, 필요 시 FCM 확장

### 5.2 Repository 패턴 예시

```java
public interface MedicineRepository {
    List<Medicine> search(String query);
    MedicineDetail getDetail(String medicineId);
}
```

Remote 실패 시 Local fallback, Local miss 시 empty/state 메시지 정책을 명확히 분리합니다.

---

## 6) 비기능 요구사항

### 성능
- 검색 응답 목표: < 1.5s (네트워크 정상 기준)
- 리스트 페이징/디바운스 적용

### 보안/개인정보
- 민감 데이터 최소 수집
- 저장 데이터 암호화(필요 시)
- 네트워크 TLS 강제

### 안정성
- API 오류/타임아웃 재시도(지수 백오프)
- 오프라인 시 읽기 전용 모드 제공

### 접근성
- 큰 글씨 대응
- 명확한 버튼 라벨/터치 영역

---

## 7) 테스트 전략

### 단위 테스트
- UseCase 로직
- ViewModel 상태 전이
- Repository fallback 로직

### 통합 테스트
- API ↔ Repository ↔ ViewModel 흐름
- 알림 스케줄 등록/취소 흐름

### UI 테스트
- 검색→상세 이동
- 처방내역 필터링
- 복약 체크 인터랙션

---

## 8) CI/CD 권장 구성

- PR마다:
  - Lint
  - Unit Test
  - Build
- main 머지 시:
  - 버전 태깅
  - APK/AAB 아티팩트 생성

예시 도구: GitHub Actions + Gradle tasks

---

## 9) 현재 코드 기준 갭 분석

현재 저장소는 기능 개요와 모듈 껍데기 정보 중심이며, 아래가 차후 구현 필요 항목입니다.

- AndroidManifest/Activity/Fragment 실제 코드
- 네트워크 클라이언트(Retrofit/OkHttp)
- 로컬 DB(Room), DAO
- 알림 스케줄러 구현
- 테스트 코드 및 CI 설정

---

## 10) 개발 환경

- IDE: Android Studio
- Language: Java
- Build: Gradle
- Min SDK: Android 6.0+ (기존 문서 기준)

---

## 11) 로드맵 제안

1. **1단계 (MVP)**: 검색 + 상세 + 기본 알림
2. **2단계**: 처방 이력 + 필터 + 설정 고도화
3. **3단계**: 데이터 품질 개선, 추천/개인화, 운영 모니터링

---

## 12) 저장소 파일 구성(현재)

```text
.
├─ README.md
└─ app/
   └─ A.java   # 초기 구조 메모
```

