---
title: "React Native 프로젝트 구조: JS 계층과 Android·iOS 네이티브 파일 이해하기"
date: 2026-08-05
tags: [react-native, expo, android, ios, gradle, cocoapods, keystore, provisioning-profile, project-structure]
description: "React Native 프로젝트가 JS 계층과 두 개의 네이티브 계층으로 나뉘는 구조를 정리하고, AndroidManifest·build.gradle·Info.plist·Podfile 등 초기 파일이 각각 무엇을 담당하는지 정리한다."
---

## 학습 목적

React Native로 앱을 시작하면 웹과 달리 **하나가 아니라 세 개의 프로젝트를 동시에 다루게 된다.** JavaScript 코드, 그리고 그 코드를 실제 앱으로 감싸는 Android 프로젝트와 iOS 프로젝트다.

```text
React Native
 ├─ JavaScript / TypeScript      화면, 상태관리, API — 두 플랫폼 공통
 ├─ Android                      실제 APK/AAB를 만드는 네이티브 프로젝트
 └─ iOS                          실제 IPA를 만드는 네이티브 프로젝트
```

평소에는 JS만 만져도 되지만, 카메라 권한을 추가하거나 앱 이름을 바꾸거나 스토어에 올리는 순간 반드시 네이티브 파일을 열게 된다. 그때 각 파일이 무엇을 담당하는지 모르면 검색한 코드를 어디에 붙여야 할지조차 판단할 수 없다.

이 글에서는 각 계층의 초기 파일이 **무엇을 결정하는 파일인지**를 정리한다.

## 먼저: 프로젝트 생성 방식에 따라 폴더가 다르다

이 부분을 모르면 "왜 내 프로젝트에는 `android/` 폴더가 없지?"에서 막힌다.

| 방식 | `android/`, `ios/` 폴더 | 네이티브 설정 방법 | 빌드 |
| --- | --- | --- | --- |
| **Expo (Managed)** | **없음** | `app.json` / `app.config.js` | EAS Build (클라우드) |
| **Expo + Prebuild** | 명령으로 **생성** | `app.json` + config plugin | 로컬 또는 EAS |
| **React Native CLI (Bare)** | 처음부터 **있음** | 파일 직접 수정 | 로컬 (Xcode / Gradle) |

Expo Managed 방식에서는 네이티브 폴더가 아예 없고, `app.json`에 적은 값을 Expo가 빌드 시점에 네이티브 설정으로 변환한다. 예를 들어 이렇게 쓰면 Android 권한과 iOS 권한 문구가 알아서 생성된다.

```json
{
  "expo": {
    "name": "My App",
    "slug": "my-app",
    "version": "1.0.0",
    "ios": {
      "bundleIdentifier": "com.example.myapp",
      "infoPlist": {
        "NSCameraUsageDescription": "프로필 사진 촬영에 사용합니다."
      }
    },
    "android": {
      "package": "com.example.myapp",
      "permissions": ["CAMERA"]
    }
  }
}
```

네이티브 폴더가 필요해지면 생성한다.

```bash
npx expo prebuild          # android/, ios/ 생성
npx expo prebuild --clean  # 기존 폴더를 지우고 다시 생성
```

여기서 **중요한 함정**이 있다. Prebuild로 생성한 폴더는 언제든 다시 만들어질 수 있는 **산출물**이다. `android/app/build.gradle`을 직접 고쳐 놓아도 다음 `prebuild --clean`에서 사라진다. Expo를 쓴다면 네이티브 설정은 파일을 직접 고치지 말고 **`app.json`이나 config plugin으로** 관리해야 한다.

아래 설명은 네이티브 폴더가 있는 상태(Bare 또는 prebuild 이후)를 기준으로 한다. Expo Managed를 쓰더라도 EAS Build가 내부에서 이 파일들을 만들어 쓰기 때문에, 구조를 알면 빌드 오류를 읽을 수 있다.

## 전체 구조

```text
my-app/
├── package.json              의존성, 스크립트
├── tsconfig.json             TypeScript 설정
├── babel.config.js           JS 변환 설정
├── metro.config.js           번들러 설정
├── app.json                  앱 이름 / Expo 설정
├── index.js                  JS 진입점
├── App.tsx                   최상위 컴포넌트
│
├── src/                      실제 앱 코드
│
├── android/                  Android 네이티브 프로젝트
│   ├── build.gradle          프로젝트 전역 빌드 설정
│   ├── settings.gradle       포함할 모듈 목록
│   ├── gradle.properties     빌드 옵션 (Hermes, 새 아키텍처 등)
│   └── app/
│       ├── build.gradle      앱 모듈 빌드 설정 (버전, 서명)
│       ├── debug.keystore    디버그 서명 키
│       └── src/main/
│           ├── AndroidManifest.xml
│           ├── java/...      MainActivity.kt, MainApplication.kt
│           └── res/          아이콘, 문자열, 스타일
│
└── ios/                      iOS 네이티브 프로젝트
    ├── Podfile               네이티브 의존성 목록
    ├── Podfile.lock          잠금 파일
    ├── MyApp.xcodeproj       Xcode 프로젝트
    ├── MyApp.xcworkspace     ← 열어야 하는 파일
    ├── Pods/                 설치된 네이티브 의존성
    └── MyApp/
        ├── Info.plist        앱 메타데이터, 권한 문구
        ├── AppDelegate.swift 앱 시작점
        └── Images.xcassets   아이콘, 스플래시
```

## JavaScript / TypeScript 계층

두 플랫폼이 공유하는 부분이며 개발 시간의 대부분을 여기서 쓴다.

### 진입점

```javascript
// index.js — 네이티브가 실행할 JS의 시작점
import { AppRegistry } from "react-native";
import App from "./App";
import { name as appName } from "./app.json";

AppRegistry.registerComponent(appName, () => App);
```

`AppRegistry.registerComponent`에 등록한 이름이 네이티브 쪽에서 찾는 이름과 일치해야 한다. Android의 `MainActivity.kt`에 있는 `getMainComponentName()`이 바로 이 값을 반환한다. **JS와 네이티브가 처음 만나는 지점**이다.

Expo Router를 쓰면 `index.js` 대신 `app/` 디렉터리의 파일 구조가 라우팅이 되므로 이 파일을 직접 볼 일이 줄어든다.

### 디렉터리 구성 예시

정해진 규칙은 없지만 다음 정도가 무난하다.

```text
src/
├── screens/        화면 단위 컴포넌트
├── components/     재사용 UI
├── navigation/     네비게이션 설정
├── api/            서버 통신
├── hooks/          커스텀 훅
├── store/          전역 상태
├── utils/          공용 함수
└── types/          타입 정의
```

### 설정 파일

| 파일 | 역할 | 언제 만지나 |
| --- | --- | --- |
| `package.json` | 의존성, 실행 스크립트 | 패키지 추가·삭제 |
| `tsconfig.json` | 타입 검사 규칙, 경로 별칭 | `@/components` 같은 별칭 설정 |
| `babel.config.js` | JS 문법 변환, 플러그인 | Reanimated, NativeWind 등 추가 시 |
| `metro.config.js` | 번들러 설정 | 모노레포, 에셋 확장자 추가 시 |
| `app.json` | 앱 이름, Expo 설정 | 앱 이름·아이콘·권한 변경 |

**Metro**는 React Native 전용 번들러다. 웹의 webpack/Vite 자리에 해당하며, JS 파일들을 하나로 묶어 앱에 전달한다. 개발 중에는 Metro 서버가 떠 있고 앱이 거기서 JS를 받아오기 때문에, JS만 고치면 다시 빌드하지 않아도 즉시 반영된다. 반대로 **네이티브 파일을 고치면 반드시 다시 빌드해야 한다.**

## Android 계층

### AndroidManifest.xml

앱의 신분증이자 명세서다. OS가 앱을 설치하고 실행할 때 이 파일을 먼저 읽는다.

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
  <!-- 필요한 권한 선언 -->
  <uses-permission android:name="android.permission.INTERNET" />
  <uses-permission android:name="android.permission.CAMERA" />

  <application
    android:name=".MainApplication"
    android:label="@string/app_name"
    android:icon="@mipmap/ic_launcher"
    android:allowBackup="false">

    <activity
      android:name=".MainActivity"
      android:launchMode="singleTask"
      android:exported="true">

      <!-- 런처에 아이콘을 표시하고 앱 시작점이 되는 설정 -->
      <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
      </intent-filter>

      <!-- 딥링크: myapp:// 로 앱 열기 -->
      <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="myapp" />
      </intent-filter>
    </activity>
  </application>
</manifest>
```

여기서 결정되는 것은 다음과 같다.

- **권한**: 카메라, 위치, 알림 등. 선언하지 않으면 런타임에 요청조차 할 수 없다.
- **앱 이름과 아이콘**: 홈 화면에 표시되는 값
- **딥링크**: 어떤 URL로 앱이 열릴지
- **화면 방향, 백업 정책** 등 앱 전역 동작

### build.gradle — 두 개가 있다

Gradle은 Android의 빌드 도구다. `build.gradle` 파일이 두 곳에 있고 역할이 다르다.

```text
android/build.gradle        ← 프로젝트 전역 (SDK 버전, 저장소)
android/app/build.gradle    ← 앱 모듈 (앱 ID, 버전, 서명)
```

**프로젝트 레벨** (`android/build.gradle`)

```gradle
buildscript {
    ext {
        minSdkVersion = 24        // 지원하는 최소 안드로이드 버전
        compileSdkVersion = 35    // 컴파일에 사용할 SDK
        targetSdkVersion = 35     // 이 버전 기준으로 동작하도록 설계했다는 선언
    }
}
```

`targetSdkVersion`은 스토어 정책과 직결된다. Google Play는 일정 수준 이상을 요구하므로, 올리지 않으면 업데이트 등록이 거부된다.

**앱 모듈 레벨** (`android/app/build.gradle`)

```gradle
android {
    namespace "com.example.myapp"

    defaultConfig {
        applicationId "com.example.myapp"   // 스토어에서 앱을 식별하는 고유 ID
        versionCode 12                      // 정수. 업로드마다 반드시 증가
        versionName "1.3.0"                 // 사용자에게 보이는 버전
    }

    signingConfigs {
        release {
            storeFile file('my-release-key.keystore')
            storePassword System.getenv("KEYSTORE_PASSWORD")
            keyAlias 'my-key-alias'
            keyPassword System.getenv("KEY_PASSWORD")
        }
    }

    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled true
        }
    }
}
```

`applicationId`는 **출시 후 변경할 수 없다.** 바꾸면 완전히 다른 앱이 되어 기존 사용자가 업데이트를 받지 못한다. 프로젝트 시작 시 신중하게 정해야 하는 값이다.

`versionCode`는 정수이며 이전보다 커야 업로드된다. `versionName`은 표시용 문자열이라 자유롭다.

### gradle.properties

빌드 옵션을 모아 둔 파일이다.

```properties
org.gradle.jvmargs=-Xmx4096m      # 빌드에 쓸 메모리. 부족하면 OOM으로 실패한다
hermesEnabled=true                 # Hermes JS 엔진 사용 여부
newArchEnabled=true                # 새 아키텍처 사용 여부
```

빌드가 메모리 부족으로 죽는다면 가장 먼저 볼 파일이다.

### Kotlin / Java 파일

```text
android/app/src/main/java/com/example/myapp/
├── MainActivity.kt
└── MainApplication.kt
```

**MainActivity.kt** — 앱의 첫 화면을 담당하는 액티비티다.

```kotlin
class MainActivity : ReactActivity() {
    // index.js의 registerComponent에 등록한 이름과 같아야 한다
    override fun getMainComponentName(): String = "MyApp"
}
```

**MainApplication.kt** — 앱 프로세스가 시작될 때 React Native 런타임을 준비한다. 네이티브 모듈 목록도 여기서 구성된다.

요즘은 **autolinking** 덕분에 라이브러리를 설치해도 이 파일을 직접 고칠 일이 거의 없다. 다만 오래된 라이브러리 문서는 여전히 수동 등록을 안내하는 경우가 있어, 이 파일이 무엇인지 알아둘 필요가 있다.

### Keystore — 가장 조심해야 할 파일

Android 앱은 **서명되지 않으면 설치되지 않는다.** 서명에 쓰는 키가 keystore다.

| 종류 | 파일 | 성격 |
| --- | --- | --- |
| 디버그 | `android/app/debug.keystore` | 모든 개발자가 공유. 저장소에 포함됨 |
| 릴리스 | 직접 생성 | **절대 저장소에 커밋하면 안 됨** |

```bash
keytool -genkeypair -v -storetype PKCS12 \
  -keystore my-release-key.keystore \
  -alias my-key-alias -keyalg RSA -keysize 2048 -validity 10000
```

핵심은 **같은 키로 서명해야 같은 앱으로 인정된다**는 점이다. 키를 잃어버리면 기존 앱의 업데이트를 올릴 수 없다.

Google Play App Signing을 사용하면 구글이 최종 서명 키를 보관하고 개발자는 업로드 키만 관리하므로, 업로드 키를 잃어버려도 재설정 요청이 가능하다. 그래도 백업은 필수다. keystore 파일과 비밀번호는 저장소가 아닌 안전한 곳에 보관하고, CI에서는 환경 변수나 시크릿으로 주입한다.

```gitignore
*.keystore
!debug.keystore
*.jks
```

## iOS 계층

### Info.plist

Android의 AndroidManifest.xml에 대응하는 파일이다. 앱 메타데이터와 권한 사용 이유를 담는다.

```xml
<dict>
  <key>CFBundleDisplayName</key>
  <string>My App</string>              <!-- 홈 화면에 보이는 이름 -->

  <key>CFBundleShortVersionString</key>
  <string>1.3.0</string>               <!-- 사용자에게 보이는 버전 -->

  <key>CFBundleVersion</key>
  <string>12</string>                  <!-- 빌드 번호. 업로드마다 증가 -->

  <!-- 권한 사용 이유. 없으면 앱이 그 자리에서 종료된다 -->
  <key>NSCameraUsageDescription</key>
  <string>프로필 사진을 촬영하기 위해 카메라를 사용합니다.</string>

  <key>NSPhotoLibraryUsageDescription</key>
  <string>프로필 사진을 선택하기 위해 사진 보관함에 접근합니다.</string>

  <!-- 딥링크 스킴 -->
  <key>CFBundleURLTypes</key>
  <array>
    <dict>
      <key>CFBundleURLSchemes</key>
      <array><string>myapp</string></array>
    </dict>
  </array>
</dict>
```

**iOS와 Android의 결정적인 차이**가 여기 있다. iOS는 권한을 요청할 때 **사용 이유 문자열이 반드시 있어야 하고, 없으면 경고가 아니라 앱이 즉시 종료된다.** 게다가 이 문구는 심사 대상이라 "권한이 필요합니다" 같은 성의 없는 문장은 반려될 수 있다.

참고로 **Bundle Identifier는 Info.plist가 아니라 Xcode 프로젝트 설정**(`PRODUCT_BUNDLE_IDENTIFIER`)에 있다. Info.plist에는 `$(PRODUCT_BUNDLE_IDENTIFIER)` 변수로 참조만 되어 있다.

### Podfile — CocoaPods

CocoaPods는 iOS의 네이티브 의존성 관리 도구다. npm이 JS 패키지를 관리한다면, CocoaPods는 그 패키지들이 포함한 **네이티브 코드**를 관리한다.

```ruby
platform :ios, '15.1'          # 지원하는 최소 iOS 버전

target 'MyApp' do
  config = use_native_modules!  # autolinking: 설치된 RN 라이브러리를 자동으로 찾는다

  use_react_native!(
    :path => config[:reactNativePath],
    :app_path => "#{Pod::Config.instance.installation_root}/.."
  )
end
```

```bash
cd ios && pod install
```

**`pod install`을 언제 해야 하는가**가 실무에서 자주 걸리는 지점이다.

- 네이티브 코드를 포함한 라이브러리를 설치·삭제했을 때
- React Native 버전을 올렸을 때
- `Podfile`을 수정했을 때
- 팀원이 위 작업을 한 커밋을 받았을 때

JS만 있는 라이브러리(예: `lodash`, `zustand`)는 `pod install`이 필요 없다. 반면 카메라·지도·결제처럼 네이티브 코드가 있는 라이브러리는 반드시 필요하다.

`Podfile.lock`은 팀 전체가 같은 버전을 쓰도록 보장하므로 **커밋해야 한다.** 반대로 `Pods/` 폴더는 보통 `.gitignore`에 넣는다.

### .xcodeproj와 .xcworkspace

```text
ios/
├── MyApp.xcodeproj      ← 이걸 열면 안 된다
└── MyApp.xcworkspace    ← 이걸 열어야 한다
```

`pod install`을 실행하면 CocoaPods가 `.xcworkspace`를 만든다. 여기에는 앱 프로젝트와 Pods 프로젝트가 함께 묶여 있다. `.xcodeproj`만 열면 Pods가 빠져서 "라이브러리를 찾을 수 없다"는 빌드 오류가 난다.

초보자가 가장 많이 겪는 iOS 빌드 실패 원인 중 하나다.

### AppDelegate

앱 생명주기의 시작점이다. React Native 런타임을 띄우고 첫 화면을 연결한다.

```text
ios/MyApp/AppDelegate.swift    # 최근 버전
ios/MyApp/AppDelegate.mm       # 이전 버전 (Objective-C++)
```

푸시 알림 등록, 딥링크 처리, 스플래시 제어처럼 **앱이 시작될 때 한 번 해야 하는 네이티브 작업**이 여기 들어간다. Android의 `MainApplication.kt` + `MainActivity.kt`에 대응한다.

### Certificate와 Provisioning Profile

iOS는 Android보다 서명 구조가 복잡하다. 실기기에 설치하거나 배포하려면 네 가지가 맞아떨어져야 한다.

| 요소 | 역할 |
| --- | --- |
| **Apple Developer 계정 (Team ID)** | 모든 것의 주체 |
| **Certificate (인증서)** | "이 개발자가 서명했다"를 증명 |
| **App ID (Bundle Identifier)** | 앱을 식별 |
| **Provisioning Profile** | 위 셋을 묶고 **어떤 기기에 설치 가능한지**를 담음 |

```text
Certificate  +  App ID  +  Device 목록
                  ↓
        Provisioning Profile
                  ↓
          앱에 서명해서 설치
```

Provisioning Profile은 용도별로 나뉜다.

| 종류 | 용도 |
| --- | --- |
| Development | 등록된 개발 기기에서 실행 |
| Ad Hoc | 지정한 기기에 테스트 배포 |
| App Store | 스토어 제출 |
| Enterprise | 사내 배포 |

**"내 아이폰에서는 되는데 팀원 기기에서 안 된다"** 는 문제는 대개 그 기기가 프로파일에 등록되지 않아서다. Ad Hoc 배포는 기기 UDID를 미리 등록해야 한다.

Xcode의 **Automatically manage signing**을 켜면 이 과정을 대신 처리해 준다. 팀 규모가 커지거나 CI에서 빌드해야 하면 수동 관리로 전환하게 되는데, 그때 이 구조를 알아야 한다. Expo EAS Build를 쓰면 인증서와 프로파일을 EAS가 대신 생성·보관해 준다.

## 두 플랫폼 개념 대응표

같은 목적의 설정이 플랫폼마다 다른 곳에 있다. 이 표만 알아도 검색이 훨씬 쉬워진다.

| 목적 | Android | iOS |
| --- | --- | --- |
| 앱 고유 식별자 | `applicationId` (build.gradle) | Bundle Identifier (Xcode 설정) |
| 표시 버전 | `versionName` | `CFBundleShortVersionString` |
| 빌드 번호 | `versionCode` | `CFBundleVersion` |
| 앱 이름 | `AndroidManifest.xml` / `strings.xml` | `CFBundleDisplayName` |
| 권한 선언 | `<uses-permission>` | `NS...UsageDescription` |
| 최소 OS 버전 | `minSdkVersion` | `platform :ios` (Podfile) |
| 의존성 관리 | Gradle | CocoaPods |
| 빌드 도구 | Gradle | Xcode Build |
| 서명 | Keystore | Certificate + Provisioning Profile |
| 배포 산출물 | `.aab` / `.apk` | `.ipa` |
| 앱 시작점 | `MainApplication` + `MainActivity` | `AppDelegate` |

## JS와 네이티브는 어떻게 연결되는가

React Native는 JS 코드를 네이티브 코드로 변환하지 않는다. **JS는 JS 엔진(Hermes)에서 그대로 실행되고, 네이티브 UI를 조작하라는 지시를 네이티브 쪽에 전달**하는 구조다.

```text
JavaScript (Hermes 엔진)
      ↕   JSI / TurboModules
네이티브 (Kotlin / Swift)
      ↓
실제 OS UI 컴포넌트
```

과거에는 이 통신이 비동기 브릿지를 통해 JSON 직렬화로 이루어져 병목이 되었다. 새 아키텍처(New Architecture)에서는 JSI를 통해 JS가 네이티브 객체를 직접 참조할 수 있어 이 비용이 줄었다.

여기서 실무적으로 중요한 결론이 나온다.

- **JS만 바꿨다** → Metro가 다시 번들링하면 끝. 앱 새로고침만 하면 된다.
- **네이티브 코드가 포함된 라이브러리를 설치했다** → `pod install`, Gradle 동기화, **앱 재빌드**가 필요하다.

"패키지를 설치했는데 `undefined is not a function`이 난다"의 흔한 원인이 재빌드 누락이다.

## 무엇을 바꿀 때 어디를 여는가

| 하고 싶은 것 | Expo Managed | 네이티브 폴더가 있을 때 |
| --- | --- | --- |
| 앱 이름 변경 | `app.json`의 `name` | `strings.xml`, `Info.plist` |
| 아이콘 변경 | `app.json`의 `icon` | `res/mipmap`, `Images.xcassets` |
| 권한 추가 | `app.json` | `AndroidManifest.xml`, `Info.plist` |
| 버전 올리기 | `app.json`의 `version` | `build.gradle`, `Info.plist` |
| 딥링크 설정 | `app.json`의 `scheme` | `intent-filter`, `CFBundleURLTypes` |
| 최소 OS 버전 | config plugin | `minSdkVersion`, `Podfile` |
| 환경별 API 주소 | `.env` + `app.config.js` | `.env` + 빌드 설정 |

## `.gitignore`에 반드시 들어가야 하는 것

```gitignore
# 빌드 산출물
android/build/
android/app/build/
ios/build/
ios/Pods/

# 서명 관련 — 유출되면 앱을 위조당할 수 있다
*.keystore
!debug.keystore
*.jks
*.mobileprovision
*.p12
*.cer

# 환경 변수
.env
.env.local

# 의존성
node_modules/
```

반대로 **`Podfile.lock`, `package-lock.json`, `gradle-wrapper.properties`는 반드시 커밋**한다. 팀원과 CI가 같은 버전으로 빌드하도록 보장하는 파일들이다.

## 자주 하는 실수

### `.xcodeproj`를 열어서 빌드

`pod install` 이후에는 `.xcworkspace`를 열어야 한다.

### 네이티브 라이브러리 설치 후 재빌드 누락

`npm install`만으로는 부족하다. iOS는 `pod install`, 양쪽 모두 앱 재빌드가 필요하다.

### iOS 권한 문구 누락

Android는 권한을 선언만 하면 되지만, iOS는 `NS...UsageDescription`이 없으면 **앱이 그 자리에서 종료된다.** 크래시 로그 없이 앱이 꺼지면 이걸 의심한다.

### `applicationId` / Bundle ID를 나중에 변경

출시 후에는 바꿀 수 없다. 프로젝트 시작 시점에 도메인 역순(`com.company.appname`)으로 신중히 정한다.

### 릴리스 keystore를 저장소에 커밋

유출되면 제3자가 같은 서명으로 앱을 위조할 수 있다. 이미 커밋했다면 히스토리에서 제거하고 키를 교체해야 한다.

### Expo prebuild 후 네이티브 파일 직접 수정

다음 `prebuild --clean`에서 되돌아간다. `app.json`이나 config plugin으로 관리한다.

### `versionCode` / `CFBundleVersion`을 올리지 않고 업로드

두 스토어 모두 같은 빌드 번호를 거부한다. 업로드마다 증가시켜야 한다.

## 정리

- React Native 프로젝트는 JS 계층 하나와 Android·iOS 네이티브 프로젝트 두 개로 구성된다.
- Expo Managed는 네이티브 폴더가 없고 `app.json`이 그 역할을 대신하며, prebuild로 생성한 폴더는 언제든 재생성되는 산출물이다.
- `AndroidManifest.xml`과 `Info.plist`가 각 플랫폼의 앱 명세서이며 권한·이름·딥링크를 결정한다.
- `build.gradle`은 프로젝트 레벨과 앱 모듈 레벨 두 개가 있고, 앱 ID·버전·서명은 앱 모듈 쪽에 있다.
- Podfile은 iOS 네이티브 의존성을 관리하며, 네이티브 라이브러리를 설치하면 `pod install`이 필요하다.
- `pod install` 이후에는 `.xcworkspace`를 열어야 한다.
- Android 서명은 keystore 하나, iOS 서명은 인증서·App ID·기기 목록을 묶은 Provisioning Profile로 이루어진다.
- 릴리스 keystore와 인증서는 저장소에 커밋하지 않고 안전하게 백업한다.
- JS 변경은 재번들링으로 끝나지만 네이티브 변경은 재빌드가 필요하다.

## 학습 체크리스트

- [ ] Expo Managed와 Bare 방식에서 네이티브 폴더의 유무와 그 이유를 설명할 수 있는가?
- [ ] `AndroidManifest.xml`과 `Info.plist`가 각각 무엇을 결정하는지 아는가?
- [ ] 프로젝트 레벨과 앱 모듈 레벨 `build.gradle`의 차이를 구분할 수 있는가?
- [ ] `applicationId`와 Bundle Identifier를 출시 후 바꿀 수 없는 이유를 아는가?
- [ ] `versionCode`와 `versionName`의 차이를 설명할 수 있는가?
- [ ] `pod install`이 필요한 상황을 판단할 수 있는가?
- [ ] `.xcodeproj`가 아니라 `.xcworkspace`를 열어야 하는 이유를 아는가?
- [ ] Certificate, App ID, Provisioning Profile의 관계를 설명할 수 있는가?
- [ ] 릴리스 keystore를 잃어버리면 어떤 문제가 생기는지 아는가?
- [ ] JS 변경과 네이티브 변경 중 언제 재빌드가 필요한지 구분할 수 있는가?

## 참고

- [React Native — 환경 설정](https://reactnative.dev/docs/set-up-your-environment)
- [React Native — Android 앱 서명 후 배포](https://reactnative.dev/docs/signed-apk-android)
- [React Native — 새 아키텍처](https://reactnative.dev/architecture/landing-page)
- [Expo — Continuous Native Generation (prebuild)](https://docs.expo.dev/workflow/continuous-native-generation/)
- [Expo — 앱 설정 (app.json / app.config.js)](https://docs.expo.dev/versions/latest/config/app/)
- [Expo — EAS Build 앱 자격 증명](https://docs.expo.dev/app-signing/app-credentials/)
- [Android Developers — 앱 매니페스트 개요](https://developer.android.com/guide/topics/manifest/manifest-intro)
- [Android Developers — 빌드 구성](https://developer.android.com/build)
- [Apple — Information Property List](https://developer.apple.com/documentation/bundleresources/information-property-list)
- [CocoaPods 사용 가이드](https://guides.cocoapods.org/using/using-cocoapods.html)
