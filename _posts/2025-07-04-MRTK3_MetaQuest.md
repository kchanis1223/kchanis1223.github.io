---
title: MRTK3 에서의 MetaQuest Passthrough 설정과 문제 해결 
date: 2025-07-02 09:13:00 +0800
categories: [Project, 게임엔진/Unity]
tags: [writing]
description: MRTK3에서 MetaQuest용 Passthrough를 설정하며 생긴 문제점과 해결방법.
render_with_liquid: false
---

## MRTK3에서 Meta Quest Passthrough 설정하기

MRTK3 초기 세팅이 완료되면 MRTK XR Rig 프리팹을 사용할 수 있다. 해당 프리팹은 XR 기반의 기본 구조를 포함하고 있으며, Meta Quest의 Passthrough 기능을 활성화하려면 아래의 추가 설정을 해주면 됨.

### 설정 방법

1. **Hierarchy**에서 `MRTK XR Rig` 프리팹을 선택  
2. 하위에 있는 **`Main Camera`** 오브젝트 선택  
3. `AR Camera Manager.cs` 스크립트를 추가  
4. 아래 코드로 Passthrough On/Off 제어 가능:

```csharp
// Passthrough 켜기
GetComponent<ARCameraManager>().enabled = true;

// Passthrough 끄기
GetComponent<ARCameraManager>().enabled = false;

```

<br>

![](https://github.com/kchanis1223/kchanis1223.github.io/blob/master/_posts/image/MRTK3Project/image05.png.png?raw=true)

<br>

## Build 이후 기기에서 실행 시 Passthrough가 나오지 않는 문제 (검은 화면).
빌드 후 Meta Quest 기기에서 Passthrough 화면이 정상적으로 나타나지 않는 경우, 아래 항목들을 순서대로 점검해보는 걸 추천.
<br>

### 점검사항.
1. **XR Plug-in Management 설정 확인**
   - `Edit > Project Settings > XR Plug-in Management`로 이동
   - Android 플랫폼에서 **OpenXR**이 활성화되어 있는지 확인
   - `OpenXR > Features` 탭에서 다음 항목이 **체크되어 있는지** 확인:
     - **Meta Quest Support**
     - **AR Passthrough
2. **AR Camera 및 Session 구성 확인**
   - `Main Camera`에 `AR Camera Manager`가 정상적으로 붙어 있는지 확인
   - `AR Session`과 `ARCameraBackGround`는 필요 없다고는 하는데, 한번 붙이고 테스트
3. **Camera Settings Manager의 디스플레이 설정 확인**
   - `Main Camera`에 연결된 **Camera Settings Manager** 컴포넌트에서 다음 설정 확인
     - **Opaque Display** : Clear Mode : Solid Color, RGBA 0,0,0,0 으로 설정
     - **Transparent Display**: Clear Mode : Solid Color, RGBA 0,0,0,0 으로 설정

<br>

### CameraSettingsManager.cs의 : DisplayType 수정
나의 위 설정을 모두 적용했음에도 **Passthrough가 출력되지 않는 문제가 발생**하였는데,  
`CameraSettingsManager.cs`의 `GetDisplayType()` 함수를 수정하여 문제를 해결할 수 있었음.

#### 수정 전 (기본 코드)

```csharp
return XRSubsystemHelpers.DisplaySubsystem.displayOpaque 
    ? DisplayType.Opaque 
    : DisplayType.Transparent;
```
<br>

수정 후 (무조건 Transparent로 강제 설정)

``` csharp
return XRSubsystemHelpers.DisplaySubsystem.displayOpaque 
    ? DisplayType.Transparent 
    : DisplayType.Transparent;
```

<br>

![](https://github.com/kchanis1223/kchanis1223.github.io/blob/master/_posts/image/MRTK3Project/image06.png.png?raw=true)

#### 오류 원인 분석

MRTK3는 실행 중 디바이스가 **Opaque(불투명 디스플레이)**인지  
**Transparent(투명 디스플레이, 예: HoloLens)**인지에 따라 `Main Camera`의 설정을 자동으로 전환하게 되어 있음.

- Opaque일 경우 → VR 환경으로 판단 → 배경을 Skybox 또는 Solid Color로 설정  
- Transparent일 경우 → AR 환경으로 판단 → 배경을 투명하게 설정해서 Passthrough가 보이도록 구성

<br>

그런데 Meta Quest는 여기서 문제가 발생함

<br>

Meta Quest는 분명 Passthrough 기능이 있는 디바이스지만,  
기기 자체가 **VR 디바이스**로 인식되기 때문에 `displayOpaque = true`로 판별됨.  
즉, MRTK 입장에서는 "이건 VR 디바이스(Opaque)"라고 판단해서  
배경을 그냥 검정색(Solid Color)으로 렌더링해버리고,  
결과적으로 **Passthrough가 전혀 보이지 않는 상태**가 되어버림.

그래서 강제로 Transparent 설정을 적용해주는 게 필요했던 것. <br>
<br>
이후에는 다른 기기를 사용하거나 시간이 지나 기기를 몇번 껐다켜니 원본 코드로도 해결이 되었음.


<br>
<br>

## Passthrough 화면이 잘려서 렌더링 되는 경우

Passthrough 기능을 On/Off 하면서 사용하다 보면,  
**화면이 일부 잘려서 렌더링되는 현상**이 발생할 때가 있음.

이 문제는 대부분 다음과 같은 이유 때문임:

- 적용한 **Cubemap**이 `Main Camera`의 **Render View Space (Far Plane)**를 벗어난 경우.

즉, Passthrough 배경으로 사용 중인 **Cubemap Material이 적용된 오브젝트**가  
**카메라의 시야 거리(Far Plane)**보다 멀리 있거나, 크기가 너무 클 경우  
일부가 카메라에 **클리핑되어 잘려 보이게 되는 것**.

![](https://github.com/kchanis1223/kchanis1223.github.io/blob/master/_posts/image/MRTK3Project/image07.png.png?raw=true)

---

### 해결 방법

- **Cubemap 오브젝트의 크기를 줄이기**,  
- `CameraSettingsManager`에서 **Far Plane Distance 값을 늘려서**  
  카메라 시야에 완전히 들어오도록 조정해주면 됨.

![](https://github.com/kchanis1223/kchanis1223.github.io/blob/master/_posts/image/MRTK3Project/image08.png.png?raw=true)
