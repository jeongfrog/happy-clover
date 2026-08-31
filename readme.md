# 행복 자판기

일상 속 작은 행복의 순간을 네잎클로버 메모로 접어 유리병에 담아두고, 힘든 날 다시 꺼내볼 수 있는 웹앱입니다.

## 주요 기능

- **로그인**: Google 계정으로 로그인해야 이용할 수 있습니다.
- **행복 담기**: 오늘 있었던 행복한 순간을 메모로 적으면 네잎클로버로 접히는 애니메이션과 함께 유리병에 저장됩니다. `#태그` 형식으로 태그를 붙일 수 있고, 담을 곳을 "내 유리병(비공개)" 또는 내가 속한 방 중에서 고를 수 있습니다.
- **행복 뽑기**: 유리병을 흔들어 담아둔 행복 메모 중 하나를 무작위로 뽑아볼 수 있습니다. 내 유리병뿐 아니라 함께 속한 방의 유리병에서도 뽑을 수 있습니다.
- **함께 나누기(방)**: 방을 만들면 6자리 참여코드가 생성됩니다. 참여코드를 아는 사람은 누구나 그 방에 입장할 수 있고(단톡방과 비슷한 구조), 방에 입장한 사람은 자신의 개인 메모 중 원하는 것만 골라 그 방으로 보내(공유해) 함께 뽑아볼 수 있습니다.
- 담긴 메모 개수만큼 유리병 안에 클로버가 채워지는 모습을 확인할 수 있습니다.

## 사용 방법

별도의 빌드 과정은 없지만, 로그인·데이터 저장을 위해 Firebase 설정이 필요합니다.

### 1. Firebase 프로젝트 준비

1. [Firebase 콘솔](https://console.firebase.google.com/)에서 새 프로젝트를 만듭니다.
2. **Authentication → Sign-in method**에서 `Google` 로그인을 사용 설정합니다.
3. **Firestore Database**를 생성합니다(프로덕션 모드 권장).
4. **프로젝트 설정 → 내 앱**에서 웹 앱을 추가하고 나오는 `firebaseConfig` 값을 복사합니다.
5. [index.html](index.html)의 `<script type="module">` 안에 있는 `firebaseConfig` 객체(`YOUR_API_KEY` 등)를 방금 복사한 값으로 교체합니다.

### 2. Firestore 보안 규칙 설정

Firebase 콘솔의 **Firestore Database → 규칙**에 아래 내용을 붙여넣고 게시합니다.

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{uid} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
      match /memos/{memoId} {
        allow read, write: if request.auth != null && request.auth.uid == uid;
      }
    }
    match /rooms/{roomId} {
      allow read: if request.auth != null;
      allow create: if request.auth != null && request.resource.data.ownerUid == request.auth.uid;
      allow update: if request.auth != null;
      match /memos/{memoId} {
        allow read: if request.auth != null
          && request.auth.uid in get(/databases/$(database)/documents/rooms/$(roomId)).data.memberUids;
        allow create: if request.auth != null
          && request.auth.uid in get(/databases/$(database)/documents/rooms/$(roomId)).data.memberUids
          && request.resource.data.authorUid == request.auth.uid;
      }
    }
  }
}
```

### 3. 로컬에서 실행

Google 로그인 팝업과 Firebase 모듈은 `file://`로 열면 동작하지 않으므로 간단한 로컬 서버로 열어야 합니다.

```bash
python3 -m http.server 8000
```

브라우저에서 `http://localhost:8000`으로 접속합니다. (localhost는 Firebase Auth에 기본으로 허용되어 있습니다.)

### 4. 배포

Firebase Hosting, Vercel, GitHub Pages 등 정적 호스팅에 그대로 배포할 수 있습니다. 배포 후에는 Firebase 콘솔의 **Authentication → Settings → 승인된 도메인**에 배포 도메인을 추가해야 Google 로그인이 동작합니다.

## 기술 스택

- HTML / CSS / Vanilla JavaScript (프레임워크·빌드 도구 없음)
- 인증: Firebase Authentication (Google 로그인)
- 데이터 저장: Firebase Firestore
- 폰트: Google Fonts (Gowun Dodum, Nanum Myeongjo)

## 데이터 구조 (Firestore)

- `users/{uid}`: 사용자 프로필(`displayName`, `email`, `photoURL`)
- `users/{uid}/memos/{memoId}`: 개인 비공개 메모(`text`, `tags`, `createdAt`, `sharedRooms`)
- `rooms/{roomId}`: 방 정보(`name`, `code`, `ownerUid`, `memberUids`, `memoCount`, `createdAt`)
- `rooms/{roomId}/memos/{memoId}`: 방에 공유된 메모(`text`, `tags`, `authorUid`, `authorName`, `createdAt`)

## 참고 사항

- 모든 메모는 Firestore에 저장되므로 로그인만 하면 어떤 기기·브라우저에서도 동일한 데이터를 볼 수 있습니다.
- 방에 보낸(공유한) 메모는 방 구성원 모두에게 공개되며, 이후 되돌리거나 삭제하는 기능은 아직 없습니다.
- `firebaseConfig`의 값(API 키 등)은 Firebase 웹 앱 특성상 클라이언트 코드에 그대로 노출되어도 되는 값이지만, 실제 데이터 접근 제어는 반드시 위의 Firestore 보안 규칙으로 이루어져야 합니다. 운영 환경에 배포하기 전 규칙을 다시 한번 검토하세요.
