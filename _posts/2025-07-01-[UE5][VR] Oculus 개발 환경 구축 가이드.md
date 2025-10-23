---
layout: post
title: "[UE5][VR] Oculus 개발 환경 구축 가이드"
description: Meta Quest 게임 개발을 위한 개발 환경 세팅 가이드
date: 2025-07-01 09:00:00 +0900
categories: [Unreal Engine, VR]
tags: [Unreal Engine, VR, XR, Oculus, MetaQuest]
---

## Oculus VR 개발환경 세팅 가이드
Oculus 환경에서 VR 게임을 개발하기 위해 필요한 엔진 설치부터 기기 세팅까지, 실제 개발에 필요한 내용을 정리해봤습니다. Oculus Quest 기기를 대상으로 VR 게임을 만들고 싶은 분들에게 도움이 될 거예요.

<br>

---

## Setup

### 1. Unreal Engine (Meta Oculus)
Oculus 기기에서 제대로 동작하는 기능들(SDK 및 전용 모듈 포함)을 사용하려면, 일반적으로 Epic Games Launcher에서 설치하는 기본 UE 버전이 아닌 [**Oculus 커스텀 버전**](https://github.com/oculus-VR/UnrealEngine/)의 언리얼 엔진이 필요합니다. 물론 플러그인 설치하고 하면 되긴 하는데, 그냥 이거 받는게 편하기도 하고, 제공하는 기능이 더 많습니다.


![feature compatibility](/assets/img/post/Oculus_Dev_Guide/UE_Installation_By_feature_compatibility.png){: width="538" height="303"}
*feature compatibility*


[Meta  공식 홈페이지](https://developers.meta.com/horizon/documentation/unreal/unreal-compatibility-matrix/)를 살펴보면 설치 엔진에 따른 특징에 대해서 나와있는데, Git Fork를 통해서 받는 경우 기본적으로 'PSO 캐시, Fast Build'와 같은 기능이 추가되어 있고, 이 외의 Depth API등을 사용할 수 있다고 하네요. 쓰시는걸 추천 드립니다..!

<br>

---

#### Engine Setup
자세한 엔진 세팅 방법은 해당 깃허브의 readme를 보면 알 수 있는데, 핵심은 아래와 같습니다. 이 과정 역시 꽤나 오래 걸립니다.

1. **깃허브에서 프로젝트 압축 해제**
2. 압축 푼 폴더 내 `Setup.bat` 실행 → 엔진 의존성 다운로드
3. `GenerateProjectFiles.bat` 실행 → `.sln` 파일 생성
4. `.sln` 열고 Build → `Development Editor`, `Win64`, `Build` 선택 후 컴파일

<br>

> GitHub 계정과 Epic Games 계정을 연동해야 해당 링크에 접근이 가능합니다. 처음이라면 [여기](https://www.unrealengine.com/en-US/ue-on-github) 를 참고하세요.
{: .prompt-info }

<br>

---

### 2. Androd Studio
Meta Quest 기기는 안드로이드 기반이기 때문에, **APK 빌드 및 배포를 위한 Android SDK/NDK** 설정이 필요합니다.

![Meta Horizon Guide](/assets/img/post/Oculus_Dev_Guide/Meta_Horizon_Guide.png){: width="636" height="282"}
*Meta Horizon Guide*

Meta Horizon Develop 페이지에 가면 위와 같은 가이드를 볼 수 있는데요. 아래 버전을 지켜가면서 설치해주면 되겠습니다.

- **SDK Build Tools**: 28.0.3 이상
- **SDK Platform**: API Level 26 이상
- **NDK 버전**: Unreal 권장 버전 사용 (ex. r25b 등)

<br>

> 이대로 설치해주시면 되는데요. 혹시나 어려우신 분들은 [Epic 공식 문서](https://dev.epicgames.com/documentation/en-us/unreal-engine/advanced-setup-and-troubleshooting-guide-for-using-android-sdk?application_version=5.4)를 참고해서 진행하시면 되겠습니다.
{: .prompt-info }

<br>

---

### 3. Meta Quest Developer Hub
빌드한 앱을 기기에 넣기 위해선 [**Meta Quest Developer Hub**](https://developers.meta.com/horizon/downloads/package/oculus-developer-hub-win)가 필요합니다. 그리고 개발자 모드 또한 활성화해주어야 알 수 없는 출처의 파일을 기기에 설치하여 테스트 할 수 있습니다.

Meta Quest Developer Hub는 아래와 같은 주요 기능들을 제공합니다.

- 실시간 디바이스 로그 확인
- 헤드셋 화면 캡처/녹화
- APK 파일 바로 기기에 설치
- Meta 스토어 업로드 연동

#### 개발자 모드 활성화
![Dev Mode](/assets/img/post/Oculus_Dev_Guide/Horizon_App.jpg){: width="663" height="340"}
*Dev Mode*

설치 이후에 Meta Horizon 어플을 통해서 기기에 대한 개발자 모드를 활성화해주어야 하는데요. 어플을 켜고, 위 그림과 같은 순서로 개발자 모드에 접근해서 활성화주면 됩니다. 

이때 처음 개발자 모드를 활성화하는 경우에는 조직을 만들라며, 링크로 연결될텐데요. 그냥 적당한 이름으로 만들어주면 됩니다.

<br>

> 역시나 자세한 내용은 [Meta Horizon Device Setup](https://developers.meta.com/horizon/documentation/native/android/mobile-device-setup/)참조 하시면 되겠습니다.
{: .prompt-tip }

<br>

---

### 4. Meta Quest Link
매번 APK를 빌드해서 기기에 넣는 건 굉장히 비효율적이죠. 그래서 사용하는 것이 [**Meta Quest Link**](https://www.meta.com/ko-kr/help/quest/1517439565442928/?srsltid=AfmBOopC3YtFxf5ky1OssNxDR2uxBJfHoedO34L05XP3h8vNE_yVRF3w)입니다.  케이블 또는 Air Link를 통해 헤드셋을 PC와 연결하고, 언리얼의 **VR Preview** 기능으로 바로 테스트할 수 있다는 장점이 있죠.

![Meta Link](/assets/img/post/Oculus_Dev_Guide/Meta_Link.png){: width="531" height="322"}
*Meta Link*

기기 연결 후, '설정 → 일반 → 알 수 없는 출처 허용' 옵션과 '개발자 런타임' 활성화해 주어야 사용 가능합니다!

<br>

![VRPreview](/assets/img/post/Oculus_Dev_Guide/VRPreview.png){: width="589" height="326"}
*VRPreview*

기기와 연결된 상태에서 Link를 활성화하고, `VR Preview`를 실행하면, 기기로 플레이가 가능해집니다. 이외에도 Meta Simulator를 사용한 더욱 편리한 방법이 있는데, 요건 나중에 따로 포스팅 해보겠습니다!

<br>

---

## 마무리
지금까지 Oculus 개발을 위한 전체적인 준비 과정을 빠르게 훑어봤습니다. 처음 세팅은 다소 번거롭지만, 개발을 위해서는 꼭 필요한 작업이죠. 이제 개발만이 남았네요. 화이팅입니다!

### 참조 링크
- [Unreal Community](https://dev.epicgames.com/community/learning/tutorials/PYP7/unreal-engine-5-5-x-for-meta-quest-vr)
- [Android Quick Start](https://dev.epicgames.com/documentation/en-us/unreal-engine/setting-up-unreal-engine-projects-for-android-development?application_version=5.4)
- [Advanced Android SDK Setup](https://dev.epicgames.com/documentation/en-us/unreal-engine/advanced-setup-and-troubleshooting-guide-for-using-android-sdk?application_version=5.4)
- [Meta Android Development Software Setup](https://developers.meta.com/horizon/documentation/native/android/mobile-studio-setup-android/?utm_source=chatgpt.com)