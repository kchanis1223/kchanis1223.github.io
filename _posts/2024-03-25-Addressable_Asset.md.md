---
title: Unity Addressable Asset
date: 2024-03-25 16:45:12 +0800
categories: [Blogging, 게임엔진/Unity]
tags: [Info]
description: Unity Addressable Asset의 개념과 사용법
render_with_liquid: false
---

# 유니티 에셋 관리 방법

유니티에서 에셋을 로드하기 위한 세 가지 주요 방법에는 리소스 폴더 사용, 에셋 번들 사용, Addressable이 있다. 
<br>
<br>
## 리소스 폴더 사용 (Resources Folder)

### 개념
- 리소스 폴더는 유니티 프로젝트 내에 있는 특정 폴더로, `Resources.Load()` 메서드를 통해 런타임 시 에셋을 동적으로 로드할 수 있음. ex) Prefab A가 있을 때, `Resources.Load<GameObject>("A")` -> 기존에 만들어 놓은 prefab가 동일한 객체 생성. <br>참고 : <https://learnandcreate.tistory.com/753>

### 장점
- 구현이 간단, 직관적.
- 런타임 시 필요한 에셋 쉽게 로드 가능.

### 단점
- 빌드 파일 크기 커짐. 
- 사용하지 않는 에셋이 포함될 수 있음.<br><br>
&rightarrow; 빌드 시간이 길어짐. 프로젝트 빌드 시 모든 에셋들은 Serialized Component로 저장되고 Meta Data가 생기는데 따라서 씬에서 사용하지 않는 오브젝트의 Meta Data도 메모리에 올라감. -> 메모리 관리 비효률적.
<br>
<br>
<br>
## 에셋 번들 사용 (Asset Bundles)

### 개념
- 에셋 번들은 에셋을 묶어놓은 것. 이를 외부 스토리지 or 클라우드에 로드하고 이후 클라이언트가 다운받아서 사용할 수 있음->원본 파일이 아니라 유니티에서 쓸 수 있도록 Serialized한 오브젝트의 헤더 정보만 로드하고 실제 요청이 들어오면 데이터를 로드하는 방식.

### 장점
- 초기 다운로드 크기를 줄일 수 있습니다.
- 플랫폼 별 최적화된 에셋을 로드할 수 있습니다.
- DLC 같은 다운로드 가능한 콘텐츠를 구현할 수 있습니다.

### 단점
- 관리가 복잡하고 시간이 많이 소요될 수 있습니다.
- 의존성 관리가 어려울 수 있습니다.

## Addressable 시스템 사용 (Addressables)

### 개념
- Addressables 시스템은 에셋을 쉽게 관리하고 런타임에 효율적으로 로드할 수 있는 방법을 제공합니다&#8203;``【oaicite:0】``&#8203;.

### 장점
- 메모리 관리가 용이하고 에셋 로드 시간이 단축될 수 있습니다.
- 콘텐츠 업데이트와 패치 적용이 쉽습니다.
- 에디터 내에서 테스트가 용이합니다.

### 단점
- 초기 설정과 구성에 시간이 소요될 수 있습니다.
- Addressable 시스템 자체의 학습 곡선이 존재합니다.
