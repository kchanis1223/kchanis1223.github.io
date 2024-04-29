---
title: Naver Map API 이용하여 지도 및 길찾기 구현하기
date: 2024-04-02 16:00:00 +0800
categories: [Project, 게임엔진/Unity]
tags: [writing]
description: 스마트 헬멧에 들어갈 기능 중, 지도와 길찾기 기능의 구현을 위해 Naver Map API 적용하기
render_with_liquid: false
---

# 목표
스마트 헬멧의 핵심 기능인 2D지도 표시와 길찾기 안내 메세지 문구 출력 기능을 구현

<br>
<br>
<br>

## 사용할 장비와 SDK
- Nreal Light : 가벼운 착용감, 아이트레킹 기능 지원,  휴대폰과 유선으로 연결됨 <br>

- NRSDK : Nreal 사에서 지원하는 SDK. 아무래도 제품을 만든 회사에서 제공하는 SDK가 가장 좋지 않을까 생각했음. <br>

- Naver Map API : Google Map , Kakao Map, Vworld map 등 지도와 길찾기 기능을 위한 여러 API가 있지만, 사용법에 대한 설명이 Naver Map이 제일 잘 되어 있는 거 같아서 일단 선택함. <br>

<br>
<br>

## Unity에 SDK , API  설치하기

- NRSDK : 홈페이지에 NRSDK에 대한 설치 방법이 자세히 설명되어 있다. <br>
링크 : <https://xreal.gitbook.io/nrsdk/nrsdk-fundamentals/quickstart-for-android> <br>
<br>

- Naver Map API 

 1. 네이버클라우드 접속 후 회원가입 진행. <br>

 2. 네이버 클라우드 콘솔창에서 AI-Naver-API에서 새로운 Application 생성. <br>

 3. 사용할 API 체크 후, 안드로이드용 Build URL 입력 후 등록. <br>

![](https://github.com/kchanis1223/kchanis1223.github.io/blob/master/_posts/image/XSaverProject/naverMap1.png?raw=true)


## Naver Map API 연결 및 자료형 확인

### 1. 2D Map

2D Map을 구현하기에는 Naver API 중, Static Map API가 가장 적절할 것 같다. 

#### 요청 형식과 파라미터

- center : string 타입 , <경도 , 위도> 입력. ex. 'center = x좌표,y좌표'
- level : int 타입 , 줌의 정도를 0~20까지 설정.
- w, h : int 타입 , 이미지의 width, height 설정. ex. 'w=크기&h=크기' 
- maptype : string 타입 , 지도 유형 설정. basic(일반) , traffic(교통 지도)
- scale : int 타입 , 해상도 설정. 1: 저해상도 ,  2: 고해상도
<br>
<br>

- 요청 헤더 : ID-KEY 인증을 해줘야 한다. <br>

![](https://github.com/kchanis1223/kchanis1223.github.io/blob/master/_posts/image/XSaverProject/naverMap2.png?raw=true)

<br>
<br>

#### C# script : MapManager
- Map parameter에 해당하는 값을 유니티에서 넣어주면 잘 동작하는 것을 확인 할 수 있다.
```c#
public class MapManager : MonoBehaviour
{
    public RawImage mapRawImage;

    [Header("Map parameter")]
    public string baseURL;
    public string APIKey = "";
    public string secretKey = "";
    public string latitude;
    public string longitude;
    public int zoomLevel = 13;
    public int mapWidth;
    public int mapHeight;
    public int scale;
    public 
    void Start()
    {
        mapRawImage = GetComponent<RawImage>();
        StartCoroutine(MapLoader());
    }
    IEnumerator MapLoader()
    {
        string sendUrl = baseURL + "?w=" + mapWidth.ToString() + "&h=" + mapHeight.ToString() +
            "&center=" + longitude + "," + latitude + "&level=" + zoomLevel.ToString() +"&scale=" + scale;

        Debug.Log(sendUrl);

        UnityWebRequest request = UnityWebRequestTexture.GetTexture(sendUrl);
        request.SetRequestHeader("X-NCP-APIGW-API-KEY-ID", APIKey);
        request.SetRequestHeader("X-NCP-APIGW-API-KEY", secretKey);

        yield return request.SendWebRequest();

        if(request.result == UnityWebRequest.Result.ConnectionError || request.result == UnityWebRequest.Result.ProtocolError)
        {
            Debug.Log(request.error);
        }
        else
        {
            mapRawImage.texture = DownloadHandlerTexture.GetContent(request);
        }
    }
}
```
<br>
<br>

![](https://github.com/kchanis1223/kchanis1223.github.io/blob/master/_posts/image/XSaverProject/naverMap3.png?raw=true)

<br>
<br>

- 인하대 하이테크의 위도,경도. 줌 레벨 15 , 고해상도, 300 * 300 pixel 설정. (위도와 경도 입력시 소수점 7자리까지 입력해야 함 )

<br>

![](https://github.com/kchanis1223/kchanis1223.github.io/blob/master/_posts/image/XSaverProject/naverMap4.png?raw=true)

<br>

### 2. Navigator Message

경로 안내 기능을 구현하기에는 Naver API 중, Direction5 가 적절한 것 같다.

#### 요청 형식과 파라미터

- startPoint , destPoint : "위도,경도" 소수점 7자리까지 표현하고 공백없이 ','로 연결하여 사용.
- navyOption : 경로 탐색 방법 설정. "traoptimal" -> 최적경로

- 지도와 마찬가지로 요청 헤더를 추가하고 사용.

#### C# script 

```c#

 IEnumerator StartNavigate()
{
    string sendUrl = navyAPIbaseURL + "?start=" + startPoint + "&goal=" + destPoint + "&option=" + navyOption;

    Debug.Log(sendUrl);

    UnityWebRequest request = UnityWebRequest.Get(sendUrl);
    request.SetRequestHeader("X-NCP-APIGW-API-KEY-ID", APIKey);
    request.SetRequestHeader("X-NCP-APIGW-API-KEY", secretKey);

    yield return request.SendWebRequest();

    if (request.result == UnityWebRequest.Result.ConnectionError || request.result == UnityWebRequest.Result.ProtocolError)
    {
        Debug.Log(request.error);
    }
    else
    {
        var text = request.downloadHandler.text;
        Debug.Log(text);
    }
}

```
<br>

 요청과 응답은 잘 되었으나, 생각치 못한 문제가 발생하였다.

<br>

![](https://github.com/kchanis1223/kchanis1223.github.io/blob/master/_posts/image/XSaverProject/naverMap5.png?raw=true)

<br>

출발지와 도착지의 위도,경도 값이 도로가 아니면 경로 탐색을 하지 못하였다. <br>

자전거의 특성상 도로가 아닌 곳에서 동작을 수행해야 될 때가 많다. <br>


-> Naver API 공식 문서를 찾아보니, 자동차 길찾기 외 기능은 추후 제공예정이라고 안내되어 있었다. <br>

따라서 길찾기 기능을 위해서는 다른 API를 쓰는것이 나을 것 같다.





