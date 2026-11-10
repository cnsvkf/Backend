---
title: "Ready GSM - domain패키지 분석"
date: 2026-09-15
tags: []
velog_url: https://velog.io/@da2un/Ready-GSM-domain%ED%8C%A8%ED%82%A4%EC%A7%80-%EB%B6%84%EC%84%9D
---

Ready GSM은 광주소프트웨어마이스터고등학교를 지원할지 고민하는 중3들이 학과 체험을 신청할 수 있게 해주는 사이트입니다.

Ready GSM server `domain` 패키지는

```
- activity
- application
- auth
- user
```

이렇게 구성되어 있습니다.
![](images/2026-09-15-Ready-GSM-domain%ED%8C%A8%ED%82%A4%EC%A7%80-%EB%B6%84%EC%84%9D/1.png)

## activity

Base path: `/api/v1/activity`

| Method | Path                          | Service                   | 설명          |
| ------ | ----------------------------- | ------------------------- | ----------- |
| GET    | `/api/v1/activity`            | QueryAllActivitiesService | 전체 활동 목록 조회 |
| GET    | `/api/v1/activity/{id}`       | QueryActivityService      | 활동 단일 조회    |
| POST   | `/api/v1/activity/admin`      | CreateActivityService     | 활동 생성       |
| PATCH  | `/api/v1/activity/admin/{id}` | EditActivityService       | 활동 수정       |
| DELETE | `/api/v1/activity/admin/{id}` | DeleteActivityService     | 활동 삭제       |

### /api/v1/activity - 전체 조회

`/api/v1/activity (GET)` - 전체 조회

`application` 도메인의 `ApplicationRepository`가 제공하는
`countApplicantsGroupedByActivity();`으로 학과 체험별 신청자 수를 한 번에 모아서 `활동 id, 신청자 수` 형태로 변환합니다.

`findAll()`로 전체 활동을 조회합니다.

* 학과 체험 이름
* 학과 체험 장소
* 학과 체험 설명
* 정원
* 날짜
  등 전체 활동을 조회합니다.
  여기에는 신청자 수 정보가 없습니다(학과 체험 자체의 정보만 있습니다.)

이러고 각 활동을 `ActivityResDto`로 변환해 `Map`을 참조해 신청자 수를 매핑한다.(Map에 값이 없으면 0으로 처리)

### /api/v1/activity/{id} - 단일 조회

`/api/v1/activity/{id}(GET)`는
`id`로 활동을 조회하고, 객체가 존재하지 않으면 404 예외를 던집니다.
존재하면 해당 활동의 신청자 수를 별도로 세어 (`countByActivity_Id`) 활동 정보와 함께 응답합니다.

### /api/v1/activity/admin - 학과 체험 생성

`/api/v1/activity/admin (POST)`는 학과 체험을 생성합니다.
`CreateActivityService` 코드를 보면
학과 체험을 생성할 때

* 학과 체험 이름
* 장소
* 학과 체험에 대한 설명
* 학과 체험 정원
* 학과 체험 날짜
* 학과 체험 신청 시작 시간
* 학과 체험 신청 마감 시간
* 활동 시작 시간, 활동 마감 시간

을 입력합니다.
마지막에 `.build()`로 앞에서 하나씩 적어나갔고 다 했다는 뜻입니다.

**build()의 역할**

```
.name(...), .place(...) 이런 것들은 "이 값 쓸게요"라고 하나씩 예약해두는 것입니다. build()를 쓰기 전까지는 ActivityJpaEntity 객체가 만들어진 게 아닙니다.
```

### /api/v1/activity/admin/{id} - 학과 체험 수정

`/api/v1/activity/admin/{id} (PATCH)`는 학과 체험 정보를 수정합니다.
먼저 `id`로 학과 체험을 조회합니다. (없으면 404 NOT_FOUND)
그리고 `activity.update(req)`를 호출해 값을 바꿉니다.
마지막으로 `countByActivity_Id`로 신청자 수를 세고
`ActivityResDto.from(activity, count)`으로 수정된 정보를 전부 합쳐 반환합니다.

그런데 여기서는 `save()`를 따로 호출하지 않았는데도 DB에 값이 반영됩니다. 그 이유는 `activity`가 `findById()`로 DB에서 직접 조회해 온 객체이고, 이 서비스가 `@Transactional` 범위 안에서 실행되기 때문입니다. JPA는 트랜잭션이 끝나는 시점에 이 객체의 값이 원래 상태와 달라졌는지 자동으로 비교하고, 달라졌으면 스스로 UPDATE 쿼리를 실행합니다(더티 체킹, Dirty Checking).

### /api/v1/activity/admin/{id} - 학과 체험 삭제

`/api/v1/activity/admin/{id} (DELETE)`는 학과 체험을 삭제합니다.
이것도 위와 같이
`deleteByActivityId(id)`를 호출해 해당 id의 활동을 바로 삭제 시도합니다. 이 메서드는 삭제된 행의 개수를 반환하는데, 그 값이 0이면(삭제할 대상이 애초에 없었다는 뜻) 404 예외를 던집니다. 별도로 조회하는 과정 없이, 삭제 쿼리 한 번의 결과만으로 존재 여부를 판단하는 방식입니다.

---

## application

Base path: `/api/v1/application`

| Method | Path                                     | Service                       | 설명              |
| ------ | ---------------------------------------- | ----------------------------- | --------------- |
| POST   | `/api/v1/application/apply`              | ApplyActivityService          | 활동 신청           |
| DELETE | `/api/v1/application/cancel`             | CancelApplicationService      | 본인 신청 취소        |
| GET    | `/api/v1/application/my`                 | QueryMyApplicationsService    | 내 신청 조회         |
| GET    | `/api/v1/application/admin/applications` | QueryApplicationsService      | (관리자) 신청자 목록 조회 |
| DELETE | `/api/v1/application/admin/cancel/{id}`  | DeleteApplicationService      | (관리자) 특정 신청 취소  |
| GET    | `/api/v1/application/admin/excel`        | ExportApplicationExcelService | (관리자) 신청자 엑셀 출력 |

학과 체험은 정원이 정해져 있어서, 정원이 다 차면 이후 신청자는 "예비(대기)"로 등록되고, 확정자가 취소하면 예비 1번이 자동으로 확정으로 승격되는 구조입니다.

### /api/v1/application/apply - 활동 신청

`/api/v1/application/apply (POST)`는 학과 체험을 신청합니다.

먼저 `activityRepository.findByIdWithLock(activityId)`로 활동을 조회합니다.

일반적인 조회가 아니라 **비관적 락(Pessimistic Lock)**이 걸린 조회입니다.

여러 학생이 동시에 마지막 한 자리를 신청할 때 정원 계산이 꼬이지 않도록, 조회하는 동안 다른 요청이 같은 활동 row를 건드리지 못하게 막는 것입니다.

활동이 없다면 `404 NOT_FOUND` 예외를 던집니다.

그 다음 `userId`로 사용자를 조회합니다.

사용자가 없다면 `404 NOT_FOUND` 예외를 던집니다.

사용자 권한이 `Role.USER`(일반 학생)인 경우에는 현재 시각이 신청 기간 안에 있는지 확인합니다.

신청 기간은

`registrationStartAt ~ registrationEndAt`

입니다.

기간 밖이라면 `400 BAD_REQUEST` 예외를 던집니다.

관리자 권한은 이 검사를 건너뜁니다.

그리고 이미 신청한 활동이 있는지 확인합니다.

`existsByUser_Id`를 사용해 이미 신청한 기록이 있다면 `409 CONFLICT` 예외를 던집니다.

그다음 현재 확정 인원 수(`isReserve = false`)와 예비 인원 수(`isReserve = true`)를 각각 셉니다.

확정 인원이 정원에 도달했다면 예비 신청으로 처리합니다.

예비로 처리해야 하는데 예비 인원이 이미 3명(`MAX_RESERVE_APPLICANT`)이라면 더 이상 예비 신청도 할 수 없기 때문에 `409 CONFLICT` 예외를 던집니다.

신청이 가능하다면 신청 정보를 빌더 패턴으로 만들어 저장합니다.

그리고 디스코드 웹훅으로 신청 알림을 보냅니다.

예비로 등록됐다면 몇 번째 대기인지 계산해서 응답에 포함합니다.

### /api/v1/application/cancel - 본인 신청 취소

`/api/v1/application/cancel (DELETE)`는 본인이 신청한 학과 체험을 취소합니다.

먼저 `userId + activityId`로 신청 내역을 조회합니다.

신청 내역이 없다면 `404 NOT_FOUND` 예외를 던집니다.

그리고 신청 기간이 이미 지났는지 확인합니다.

신청 기간이 지났다면 취소할 수 없기 때문에 `400 BAD_REQUEST` 예외를 던집니다.

그다음 취소하기 전에 이 신청이 확정자였는지(`!isReserve()`) 기억해둡니다.

신청 내역을 삭제합니다.

그리고 방금 삭제한 신청이 확정자였다면 예비 인원 중 가장 먼저 신청한 사람을 찾습니다.

```java
findFirstByActivity_IdAndIsReserveTrueOrderByCreatedAtAscIdAsc
```

이 메서드를 사용해서 가장 먼저 신청한 예비 신청자를 찾습니다.

그리고 `promote()`를 호출해 확정으로 승격시킵니다.

즉, 확정 자리가 하나 비면 예비 1번이 자동으로 확정되는 구조입니다.

마지막으로 디스코드로 취소 알림을 보냅니다.

### /api/v1/application/my - 내 신청 조회

`/api/v1/application/my (GET)`는 내가 신청한 학과 체험을 조회합니다.

먼저 `userId`로 신청 내역을 조회합니다.

신청 내역이 없다면 `404 NOT_FOUND` 예외를 던집니다.

그리고 예비 인원이라면 나보다 먼저 신청한 예비 인원 수(`countReserveApplicantsAheadOf`)를 세어 내 대기 순번을 계산합니다.

### /api/v1/application/admin/applications - 관리자용 신청자 목록

`/api/v1/application/admin/applications (GET)`은 관리자가 해당 활동의 신청자 목록을 조회할 때 사용하는 API입니다.

먼저 해당 활동의 전체 신청 내역을 조회합니다.

그리고 예비 인원들만 골라 `생성 시각 → id` 순으로 정렬합니다.

정렬한 순서대로 각자에게 대기 순번을

`1, 2, 3...`

형태로 매깁니다.

그리고 전체 신청 목록을 DTO로 변환하면서 예비 인원에게는 위에서 계산한 순번을 붙입니다.

### /api/v1/application/admin/cancel/{id} - 관리자 강제 취소

`/api/v1/application/admin/cancel/{id} (DELETE)`는 관리자가 특정 신청을 강제로 취소할 때 사용하는 API입니다.

`CancelApplicationService`(본인 취소)와 흐름은 거의 같습니다.

신청을 삭제하고, 삭제한 신청이 확정자였다면 예비 1번을 자동으로 확정시킵니다.

다만 본인 취소와 다르게 신청 기간 검사와 디스코드 알림은 없습니다.

관리자 작업이므로 신청 기간과 관계없이 언제든 처리할 수 있도록 만든 것입니다.

### /api/v1/application/admin/excel - 엑셀 출력

`/api/v1/application/admin/excel (GET)`은 관리자가 신청자 목록을 엑셀로 출력할 수 있게 해주는 API입니다.

먼저 활동과 해당 활동의 전체 신청자를 조회합니다.

그리고 Apache POI 라이브러리를 사용해서 엑셀 워크북을 생성합니다.

엑셀 시트는

* 신청자 목록
* 신청 대기자 목록

두 개로 나눕니다.

그리고 `isReserve` 값에 따라서 각 신청자를 알맞은 시트에 한 줄씩 기록합니다.

완성된 워크북을 byte 배열로 변환합니다.

그리고 `활동명 + 타임스탬프`로 파일명을 만듭니다.

마지막으로 Controller에서 이 byte 배열을 응답 바디에 담고,

`Content-Disposition: attachment`

헤더를 설정합니다.

이렇게 하면 브라우저에서 해당 응답을 파일로 다운로드할 수 있습니다.

---

## auth

Base path: `/api/v1/auth`

| Method | Path                      | Service                    | 설명        |
| ------ | ------------------------- | -------------------------- | --------- |
| POST   | `/api/v1/auth/logout`     | LogoutService              | 로그아웃      |
| POST   | `/api/v1/auth/{provider}` | OAuthAuthenticationService | OAuth 로그인 |

인증 방식은 **세션 기반**입니다.

JWT를 사용하지 않고, 로그인에 성공하면 인증 정보를 `HttpSession`에 저장해둡니다.

그 이후 요청은 세션 쿠키를 사용해서 로그인 상태를 유지합니다.

### /api/v1/auth/logout - 로그아웃

`/api/v1/auth/logout (POST)`는 로그아웃을 처리합니다.

먼저 현재 요청의 `HttpSession`을 가져옵니다.

세션이 존재하면 `invalidate()`를 호출해서 세션을 무효화합니다.

그리고 `SecurityContextHolder.clearContext()`로 인증 정보를 지웁니다.

### /api/v1/auth/{provider} - OAuth 로그인

`/api/v1/auth/{provider} (POST)`는 OAuth를 사용해서 로그인을 처리합니다.

프론트엔드가 OAuth 제공자(카카오, 구글)로부터 받은 인가 코드(Authorization Code)와 `redirect_uri`를 서버로 넘기면 서버가 이를 검증해서 로그인을 완료해주는 방식입니다.

먼저 `redirectUri`가 비어있는지 확인합니다.

그리고 서버에 등록된 허용 목록(`allowedRedirectUris`)에 해당 `redirectUri`가 있는지 확인합니다.

허용 목록에 없는 URL이라면 예외를 던집니다.

이렇게 하는 이유는 아무 URL로나 로그인 결과가 리다이렉트되는 것을 막기 위해서입니다.

그다음 `OAuthProviderFactory`에서 `{provider}` 값에 맞는 `OAuthProvider` 구현체를 가져옵니다.

예를 들어 `{provider}`가 `kakao`라면 카카오 OAuth를 처리하는 구현체를 가져옵니다.

이 구현체가 실제로 OAuth 서버와 통신해서 인가 코드를 사용하고 사용자 이메일 등의 정보를 받아옵니다.

그다음 받아온 이메일과 가입 경로(`authReferrerType`)를 사용해서 DB에서 기존 회원을 찾습니다.

회원이 존재하지 않는다면 신규 회원으로 저장합니다.

즉, 최초 로그인이라면 자동으로 회원가입까지 처리됩니다.

그리고 사용자의 마지막 로그인 시각을 갱신합니다.

그다음 `OAuth2AuthenticationToken`을 만들어 `SecurityContext`에 설정합니다.

기존 세션이 있다면 무효화하고, 새 세션을 만들어 방금 만든 `SecurityContext`를 저장합니다.

---

## user

Base path: `/api/v1/user`
개발용 API는 `/api/v1/utility`를 사용합니다.

| Method | Path                        | Service               | 설명              |
| ------ | --------------------------- | --------------------- | --------------- |
| GET    | `/api/v1/user/me`           | QueryUserService      | 내 정보 조회         |
| PATCH  | `/api/v1/utility/user/role` | ModifyUserRoleService | (개발용) 사용자 권한 변경 |

### /api/v1/user/me - 내 정보 조회

`/api/v1/user/me (GET)`은 현재 로그인한 사용자의 정보를 조회합니다.

`@AuthenticationPrincipal`을 사용해서 세션에 저장되어 있는 `OAuth2User` 객체를 그대로 받습니다.

그리고 이 객체의 attribute에서

* `id`
* `email`
* `role`

을 꺼냅니다.

꺼낸 정보를 `UserResDto`로 만들어 반환합니다.

여기서는 DB 조회가 전혀 없습니다.

로그인할 때 필요한 사용자 정보를 세션에 저장해두었기 때문에 이 API에서는 DB를 다시 조회하지 않고 세션에 저장되어 있는 값을 그대로 사용합니다.

그래서 이 API는 매우 가볍게 동작합니다.

### /api/v1/utility/user/role - 사용자 권한 변경

`/api/v1/utility/user/role (PATCH)`는 개발할 때 테스트 계정의 권한을 변경하기 위한 API입니다.

`UtilityController`에는 `@Profile("!prod")`가 붙어 있습니다.

그래서 운영 환경인 `prod`에서는 이 API 자체가 등록되지 않습니다.

로컬이나 개발 환경에서 테스트 계정의 권한을 쉽게 바꾸기 위한 유틸리티입니다.

먼저 이메일로 사용자를 조회합니다.

사용자가 없다면 `404 NOT_FOUND` 예외를 던집니다.

사용자가 존재한다면 `user.modifyRole(role)`을 호출해서 권한을 변경합니다.

여기서도 activity 수정 API와 같은 원리로 `save()`를 따로 호출하지 않습니다.

`@Transactional` 범위 안에서 DB에서 조회한 엔티티의 값을 변경했기 때문에 트랜잭션이 끝날 때 JPA의 **더티 체킹(Dirty Checking)**으로 변경된 값이 DB에 자동으로 반영됩니다.

---

## 도메인 간 협력 관계 정리

### application -> activity

`application` 도메인은 학과 체험 신청을 담당하고 `activity` 도메인은 학과 체험 자체를 담당합니다.

신청을 생성하거나 취소할 때 어떤 학과 체험인지 알아야 하기 때문에 `application` 도메인에서는 `activity`를 참조합니다.

신청이 생성되거나 취소되면 해당 학과 체험의 신청자 수와 남은 정원에 영향을 줍니다.

### activity -> application

`activity`의 조회 API에서는 신청자 수가 필요합니다.

그래서 `activity`의 전체 조회와 단일 조회, 수정 API에서 `application` 도메인의 `ApplicationRepository`를 사용해 신청자 수를 조회합니다.

즉, 현재 구조에서는 `activity` 도메인도 `application` 도메인의 Repository에 의존하고 있습니다.

### application -> user

`application` 도메인에서는 신청한 사용자가 누구인지 알아야 하기 때문에 `user` 엔티티를 참조합니다.

신청할 때 `userId`로 사용자를 조회하고, 이미 해당 활동에 신청했는지도 확인합니다.

### auth -> user

`auth` 도메인에서는 OAuth 로그인을 처리합니다.

OAuth 로그인이 성공하면 OAuth 제공자로부터 받은 이메일과 가입 경로를 사용해서 `user`에서 기존 회원을 찾습니다.

회원이 없다면 새로운 회원으로 저장합니다.

즉,

**OAuth 로그인 -> 회원 조회 -> 없으면 회원가입 -> 세션에 인증 정보 저장**

순서로 동작합니다.

### 인증 정보 -> user

로그인에 성공하면 인증 정보를 `HttpSession`에 저장합니다.

그 후 `/api/v1/user/me`나 `@AuthRequest`에서는 매번 DB에서 사용자를 조회하지 않고 세션에 저장해둔 `OAuth2User` 정보를 사용합니다.

그래서 로그인한 사용자를 식별할 때 DB를 다시 조회하지 않고 세션에 저장된 정보를 재사용할 수 있습니다.

