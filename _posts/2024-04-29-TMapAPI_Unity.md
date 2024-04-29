---
title: TMap API을 이용하여 지도 및 길찾기 구현하기
date: 2024-04-29 10:12:34 +0800
categories: [Project, 게임엔진/Unity]
tags: [writing]
description: 스마트 헬멧에 들어갈 기능 중, 지도와 길찾기 기능의 구현을 위해 TMap API 적용하기
render_with_liquid: false
---

# 목표
스마트 헬멧의 핵심 기능인 2D지도 표시와 길찾기 안내 메세지 문구 출력 기능을 구현<br>
-> Naver Map API에서는 아직 자동차 길찾기에 도움이 되는 API만 제공이 되었다. 길찾기 API를 찾아보니 TMAP API가 있었고, 도보 및 자전거를 위한 길찾기 정보를 제공하는 API가 있었다. 


## TMap API

1. SK open API에 접속 후 회원가입. 링크 : <https://openapi.sk.com/> <br>

2. 대시보드 > 앱 > 새로운 앱 추가. Tmap을 위한 앱 등록하기 <br>

3. 새로 만든 앱의 appKey 확인하기.

<br>
![](https://github.com/kchanis1223/kchanis1223.github.io/blob/master/_posts/image/XSaverProject/tmap1.png?raw=true)

4. 사용 신청 하기. (이거 안한채로 계속 request 하다가 appkey 인증이 안되서 계속 실패했음.. curl을 정확히 작성했는데도 403 Forbidden Error가 생긴다면 다시 이 부분을 다시 확인해보자 )

<br>
![](https://github.com/kchanis1223/kchanis1223.github.io/blob/master/_posts/image/XSaverProject/tmap2.png?raw=true)

### 1. 2D Map

요청 형식과 파라미터

- version (필수) : 호출할 API의 버전. 기본값 1
- latitude (필수) : 경도 값. 소수점 11자리까지 표현가능
- longitude (필수) : 위도 값. 소수점 11자리까지 표현 가능
- zoom (필수) : 9~20까지 줌의 정도를 설정.
- width, height (필수) : 받아올 이미지 크기 설정.
- markers : 해당 좌표에 마킹 할 수 있음.
- format : 받아올 이미지의 파일 형식 설정.

<br>

- Naver API와 마찬가지로 key인증을 해주어야 한다. <br>

![](https://github.com/kchanis1223/kchanis1223.github.io/blob/master/_posts/image/XSaverProject/tmap3.png?raw=true)

<br>
C# script

```c#
 private string APIKey = "<인증을 위한 appkey 값>";

 [Header("Map parameter")]
 private string mapAPIbaseURL = "https://apis.openapi.sk.com/tmap/staticMap";
 private string coordType = "WGS84GEO";
 private string format = "PNG";
 public string latitude = "37.5446283608815"; // 소수점 11자리
 public string longitude = "126.83529138565";
 public int zoomLevel = 15;
 public int mapWidth = 512;
 public int mapHeight = 512;
 public string marker = "126.978155,37.566371";

  IEnumerator MapLoader()
 {
     string sendUrl = mapAPIbaseURL +
         "?version=1" +
         "&coordType=" + coordType +
         "&width=" + mapWidth + 
         "&height=" + mapHeight +
         "&zoom=" + zoomLevel +
         "&format=" + format +
         "&longitude=" + longitude +
         "&latitude=" + latitude +
         "&markers=" + marker;

     using (UnityWebRequest request = UnityWebRequestTexture.GetTexture(sendUrl))
     {
         request.SetRequestHeader("accept", "application/json");
         request.SetRequestHeader("appKey", APIKey);
         yield return request.SendWebRequest();

         if (request.result == UnityWebRequest.Result.ConnectionError || request.result == UnityWebRequest.Result.ProtocolError)
         {
             Debug.Log(request.error);
         }
         else
         {
             Texture2D texture = DownloadHandlerTexture.GetContent(request);
             if (mapRawImage != null) mapRawImage.texture = texture;
             else Destroy(texture);
         }
     }
 }
```
<br>
<br>

### 실행화면 

<br>

![](https://github.com/kchanis1223/kchanis1223.github.io/blob/master/_posts/image/XSaverProject/tmap4.png?raw=true)

<br>
<br>

### 2. Navigation
자전거 경로의 특성상, 자동차의 경로 설정보다는, 도보의 경로 설정을 이용하는 것이 더 알맞다고 생각하여 routes/pedestrian을 이용하였다. <br> <br>

요청 형식과 파라미터

- startX, startY (필수) : 출발지 x 좌표 -> 경도
- endX, endY (필수): 출발지 y 좌표 ->위도
- startName, endName (필수): 출발지, 목적지의 명칭. UTF-8기반 URL 인코딩 해야함.
- searchOption : 경로 탐색 옵션값 0-추천 , 4-추천+대로우선 , 10-최단 , 30-최단+계단제외
- speed : 진행속도 (km/h)
- angle : 각도
- passlist: 경유지를 x,y로 나타내고 나열할 수 있음.
- sort : 지리정보 개체의 정렬 순서를 지정.

<br>
<br>

#### 2D 지도의 경우와 마찬가지로 appkey 인증을 헤더에 추가 해야한다.

<br>
<br>

```c#
IEnumerator StartNavigate()
{
    string sendUrl = navyAPIbaseURL + "?version=1";
    float startPoint_X = 126.92365493654832F;
    float startPoint_Y = 37.556770374096615F;
    float endPoint_X = 126.92432158129688F;
    float endPoint_Y = 37.55279861528311F;
    string startName = "%EA%B4%91%EC%B9%98%EA%B8%B0%ED%95%B4%EB%B3%80"; // url 인코딩 해야됨
    string endName = "%EC%98%A8%ED%8F%89%ED%8F%AC%EA%B5%AC";
    var navyjson = new
        {
            startName = startName,
            startX = startPoint_X.ToString(),
            startY = startPoint_Y.ToString(),
            endName = endName,
            endX = endPoint_X.ToString(),
            endY = endPoint_Y,
            searchOption = "0",
            sort = "custom"
        };
    
    string str_navyjson = JsonConvert.SerializeObject(navyjson);

    using (UnityWebRequest request = UnityWebRequest.Post(sendUrl, "POST"))
    {
        byte[] jsonToSend = new System.Text.UTF8Encoding().GetBytes(str_navyjson);
        request.uploadHandler = (UploadHandler)new UploadHandlerRaw(jsonToSend);
        request.downloadHandler = (DownloadHandler)new DownloadHandlerBuffer();
        request.SetRequestHeader("accept", "application/json");
        request.SetRequestHeader("Content-Type", "application/json");
        request.SetRequestHeader("appKey", APIKey);

        yield return request.SendWebRequest();

        if (request.result == UnityWebRequest.Result.ConnectionError || request.result == UnityWebRequest.Result.ProtocolError)
        {
           
            Debug.Log($"Error: {request.error}");
            Debug.Log($"Response: {request.downloadHandler.text}");
            Debug.Log($"Response Code: {request.responseCode}");
            
        }
        else
        {
            Debug.Log($"Successful Response: {request.downloadHandler.text}");
        }
    }
}
```
<br>

- Json 데이터를 Post로 넘겨주고 경로 안내에 해당하는 Json 데이터를 받음.

<br>
<br>

### 실행결과
<br>

![](https://github.com/kchanis1223/kchanis1223.github.io/blob/master/_posts/image/XSaverProject/tmap5.png?raw=true)

<br>
<br>

request의 데이터를 Json 타입으로 받아와 저장하려고 하니, 데이터를 파싱해서 미리 정의된 변수에 저장하는 과정에서, 타입이 맞지 않는 경우가 꽤 발생하는 문제가 발생하였다. <br>
1. Error : "ArgumentException: Could not cast or convert from System.Double to System.Collections.Generic.List`1[System.Double]." <br>
Sol.) public class Geometry의 coordinates를 object 타입으로 설정.
<br>
<br>

2. Error : "InvalidCastException: Null object cannot be converted to a value type." <br>
Sol.) public class Properties의 int 타입으로 설정하였던 roadType, categoryRoadType, facilityType, turnType을 int?로 변경.
<br>
<br>

```c#

    // json data 정의
    [System.Serializable]
    public class NavyJson
    {
        public string type;
        public List<Feature> features;
    }

    [System.Serializable]
    public class Feature
    {
        public string type;
        public Geometry geometry;
        public Properties properties;
    }

    [System.Serializable]
    public class Geometry
    {
        public string type;
        public object coordinates;
    }

    [System.Serializable]
    public class Properties
    {
        public int index;
        public int lineIndex;
        public string name;
        public string description;
        public float distance;
        public float time;
        public int? roadType;
        public int? categoryRoadType;
        public int? facilityType;
        public string facilityName;
        public string direction;
        public string nearPoiName;
        public string nearPoiX;
        public string nearPoiY;
        public string intersectionName;
        public int? turnType;
        public string pointType;
    }
```
추가로 기존 코드 try and catch로 수정하고 NavyJson에 저장하여 출력하였다.
<br>
<br>

```c#
else
{
    try
    {
        NavyJson navyjson = JsonConvert.DeserializeObject<NavyJson>(request.downloadHandler.text);
        ProcessFeatures(navyjson);
    }
    catch(JsonSerializationException ex)
    {
        Debug.LogError("Json Serialized Error :" + ex.Message);
    }
    catch (Exception ex)
    {
        Debug.LogError("General Error : " + ex.Message);
    }
    
}
```
<br>

```c#
void ProcessFeatures(NavyJson navyjson)
{
    foreach(var feature in navyjson.features) Debug.Log($"Feature Name: {feature.properties.name}, Description: {feature.properties.description}");
}
```
<br>

실행화면

<br>

![](https://github.com/kchanis1223/kchanis1223.github.io/blob/master/_posts/image/XSaverProject/tmap6.png?raw=true)
