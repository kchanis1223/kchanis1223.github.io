---
title: TMap API을 이용하여 길찾기 메세지 & 속도 표시
date: 2024-07-01 09:41:34 +0800
categories: [Project, 게임엔진/Unity]
tags: [writing]
description: 스마트 헬멧에 들어갈 기능 중, 길찾기 기능 & 속도 표기 기능 구현하기
render_with_liquid: false
---

# 목표
현재 위치에 맞는 길 안내 메세지 출력 및 현재 속도 표기<br>
-> 속도 표기 : Android의 GPS + Haversine Distance. <br>
-> 길 안내 : 앞서 구한 Tmap api json 데이터 + GPS 현재 위치 매핑. <br><br>


##  속도 표기
속도 표기는 크게 다음의 순서로 동작한다. <br>

1. GPS 권한 설정 <br>
2. 위도&경도 받아오기 <br>
3. 현재 위치 - 이전 위치 비교를 통해 속도 계산<br>
4. AR HUD에 표시 <br>
5. n초 마다 업데이트 <br>


### Android GPS 권한 설정 & GPS 정보 받아오기

StartGPS() 와 GPSManager() 를 만들고, Start()에서 코루틴으로 동작하도록 하였다. <br>
StartGPS()는 GPS에 연결과 초기화를 하고, 오류에 대한 디버그 메세지를 출력함 <br>
GPSManager()는 권한 설정 한 뒤, 매초마다 위치 값 얻어옴 <br>

<br>
<br>

StartGPS() 스크립트

```c#
IEnumerator StartGPS()
{
    // 위치 서비스가 시작되지 않았거나 꺼져 있을 때
    if (!Input.location.isEnabledByUser)
        yield break;

    // 위치 서비스 시작
    Input.location.Start();

    // 위치 서비스가 초기화될 때까지 대기
    int maxWait = 20;
    while (Input.location.status == LocationServiceStatus.Initializing && maxWait > 0)
    {
        yield return new WaitForSeconds(1);
        maxWait--;
    }

    // 초기화에 실패했을 때
    if (maxWait < 1)
    {
        Debug.Log("Timed out");
        yield break;
    }

    // 서비스 사용 불가 상태
    if (Input.location.status == LocationServiceStatus.Failed)
    {
        Debug.Log("Unable to determine device location");
        yield break;
    }
    else
    {
        // 첫 위치 설정
        lastPosition = new Vector3(Input.location.lastData.latitude, Input.location.lastData.longitude, 0);
        isFirstUpdate = true;

        // 위치 업데이트 코루틴 시작
        StartCoroutine(UpdateLocation());
    }
}
```
* UpdateLocation()를 호출하여 속도계산과 업데이트 주기를 정해준다. <br>

* 연동 실패시, 디버그 메세지에 대한 수정을 구글링하고 하면 된다!
<br>
<br>
<br>

GPSManager() 스크립트


```c#
IEnumerator GPSManager()
{
    if (!Permission.HasUserAuthorizedPermission(Permission.FineLocation))
    {
        Permission.RequestUserPermission(Permission.FineLocation);
        while (!Permission.HasUserAuthorizedPermission(Permission.FineLocation))
        {
            yield return null;
        }
    }

    if (!Input.location.isEnabledByUser)
    {
        InfoText[0].gameObject.SetActive(true);
        InfoText[0].text = "GPS OFF";
        yield break;
    }

    Input.location.Start();
    while (Input.location.status == LocationServiceStatus.Initializing && GPS_delay < max_delay_time)
    {
        yield return new WaitForSeconds(1.0f);
        GPS_delay++;
    }

    if (Input.location.status == LocationServiceStatus.Failed || Input.location.status == LocationServiceStatus.Stopped)
    {
        InfoText[0].gameObject.SetActive(true);
        InfoText[0].text = "Delay Time Over";
        yield break;
    }

    while (true)
    {
        latitude = Input.location.lastData.latitude.ToString("F7");
        longitude = Input.location.lastData.longitude.ToString("F7");
        marker = longitude + "," + latitude;
        InfoText[0].text = "latitude : " + latitude + ", longitude : " + longitude;

        StartCoroutine(MapLoader(latitude, longitude, marker));
        yield return new WaitForSeconds(2.0f);
    }
}
```
<br>

* 정상적으로 작동하면 InfoText[0] 에 위경도가 표기됨. <br>
이때,  .ToString("F7) 으로 위경도의 표현 범위를 제한하였는데 이후의 MapLoader()에서 사용될 (Tmap API의 curl 요청 형식에 맞추기 위함) 위경도에 형식이 중요하기 떄문이다.<br>
-> 일반적인 안드로이드 휴대폰 GPS의 경우 소수점 5~6 자리 까지 표기 해주는데(11cm ~ 1.1m 정확도), .ToString("F7")을 하면 이후는 0으로 표기된다.<br>
<br>

* while(true) 와 yield return new WaitForSeonds(n) 으로 지도표기의 업데이트 주기를 정해주면 된다. 속도 표기만 생각한다면, MapLoader() 부분은 지우고 쓰면 된다.

<br>
<br>

### 속도 계산&표기 및 업데이트

UpdateLocation()에서 현재 속도 계산 과 GPS값 업데이트 주기를 결정하고 HaversineDistance() 에서 움직인 거리를 계산한다. 
<br>
<br>
<br>
UpdateLocation() 스크립트

```c#
IEnumerator UpdateLocation()
{
    while (true)
    {
        // 현재 위치
        Vector3 currentPosition = new Vector3(Input.location.lastData.latitude, Input.location.lastData.longitude, 0);

        if (!isFirstUpdate)
        {
            // 실제 시간 간격 계산
            float deltaTime = Time.time - lastUpdateTime;

            // 거리 계산 (Haversine formula)
            float distance = HaversineDistance(lastPosition.x, lastPosition.y, currentPosition.x, currentPosition.y);

            // threshold 설정
            if (distance >= MinMovement)
            {
                // 속도 계산 (거리 / 시간)
                float speed = distance / deltaTime;
                float speedKmh = speed * 3.6f;
                // 속도 표시
                velocityText.text = speed.ToString("F1") + " km/h";

                // 마지막 위치와 시간 업데이트
                lastPosition = currentPosition;
                lastUpdateTime = Time.time;
            }
        }
        else
        {
            isFirstUpdate = false;
            lastPosition = currentPosition;
            lastUpdateTime = Time.time;
        }

        // 위치 업데이트 간격
        yield return new WaitForSeconds(1);
    }
}
```
* 기기의 GPS 성능에 따라 노이즈와 이상치가 발생하는 경우가 있었는데, 1) 움직이지 않았는데 GPS값이 바뀐경우 & 2) 조금 움직였는데 GPS 값이 너무 크게 바뀐 경우. 이와 같은 경우를 고려하여 MinMovement와 MaxMovement를 두어 정상적인 값을 골라내는것이 필요하였다. <br>
MinMovement로 1번 문제를 해결하고 MaxMovement로 2번 문제를 해결할 수 있다. (코드는 1번만 적용)
<br>
<br>
<br>
HaversineDistance()
<br>

```c#
private float HaversineDistance(float lat1, float lon1, float lat2, float lon2)
{
    float R = 6371000; // 지구 반지름
    float dLat = (lat2 - lat1) * Mathf.Deg2Rad;
    float dLon = (lon2 - lon1) * Mathf.Deg2Rad;
    float a = Mathf.Sin(dLat / 2) * Mathf.Sin(dLat / 2) +
              Mathf.Cos(lat1 * Mathf.Deg2Rad) * Mathf.Cos(lat2 * Mathf.Deg2Rad) *
              Mathf.Sin(dLon / 2) * Mathf.Sin(dLon / 2);
    float c = 2 * Mathf.Atan2(Mathf.Sqrt(a), Mathf.Sqrt(1 - a));
    float distance = R * c;
    return distance;
}
```
* Harversine Distance는 위경도 좌표 사이의 거리를 구할 떄 쓰는 방법인데, 자세한 구현은 구글링하면 잘 나와있어서 참고하여 작성하였다. 
<br>
<br>


## 2. 길 안내 메세지 표기

길안내 메세지는 다음 순서대로 동작하도록 하였다.<br>
1. 사용자 목적지 입력받기 <br>
2. 입력받은 목적지 위경도값으로 변환 ( Text -> long+lat) 
3. TMAP API 보행자 경로 탐색 
4. 길 안내 Json 데이터 파싱 및 저장
5. 현재 위치에 해당하는 메시지 계산
6. 메세지 출력 및 업데이트
<br>
<br>

### 사용자 목적지 입력받고 변환하기
Unity의 InputField를 이용하여 사용자에게 목적지를 텍스트로 입력받고, TMAP API의 Geocoding API를 이용하여 앞서 사용한 다른 API의 curl 방식과 동일한 방법으로 목적지 텍스트에 해당하는 위경도값을 구할 수 있다. 
<br>
<br>

SK open API Geocoding 링크 : `https://openapi.sk.com/products/detail?linkMenuSeq=24` <br>

<br>
<br>

### 길 안내 데이터 받아오고 저장하기
StartNavigate()에서는 API를 호출하고 받은 데이터를 파싱하고 저장한다. 정상적으로 Json 데이터를 처리하였다면 UpdateNavigation()을 호출한다.

<br>
<br>
StartNavigate() 스크립트

```c#
IEnumerator StartNavigate(string latitude, string longitude, string destination)
{
    // 37.4506600  126.6576000
    string sendUrl = navyAPIbaseURL + "?version=1";
    string startPoint_X = longitude;
    string startPoint_Y = latitude;
    string endPoint_X = Text2GPS(destination).X;
    string endPoint_Y = Text2GPS(destination).Y;

    string endName = destination;

    
    var request_json = new
    {
        startName = startName,
        startX = startPoint_X,
        startY = startPoint_Y,
        endName = endName,
        endX = endPoint_X,
        endY = endPoint_Y,
        searchOption = "0",
        sort = "custom",
        reqCoordType = "WGS84GEO"
    };

    string str_navyjson = JsonConvert.SerializeObject(request_json);

    using (UnityWebRequest request = new UnityWebRequest(sendUrl, "POST"))
    {
        byte[] jsonToSend = Encoding.UTF8.GetBytes(str_navyjson);
        request.uploadHandler = new UploadHandlerRaw(jsonToSend);
        request.downloadHandler = new DownloadHandlerBuffer();
        request.SetRequestHeader("accept", "application/json");
        request.SetRequestHeader("Content-Type", "application/json");
        request.SetRequestHeader("appKey", APIKey);
        yield return request.SendWebRequest();

        if (request.result == UnityWebRequest.Result.ConnectionError || request.result == UnityWebRequest.Result.ProtocolError)
        {
            Debug.LogError($"Error: {request.error}");
            Debug.LogError($"Response: {request.downloadHandler.text}");
            Debug.LogError($"Response Code: {request.responseCode}");
        }
        else
        {
            try
            {
                string jsonResponse = request.downloadHandler.text;
                Debug.Log("JSON Response: " + jsonResponse); // JSON 응답 출력

                // JSON 데이터를 동적으로 로드
                JObject json = JObject.Parse(jsonResponse);

                // NavyJson 객체로 변환
                currentRoute = json.ToObject<NavyJson>();
                ProcessFeatures(currentRoute);

                StartCoroutine(UpdateNavigation());
            }
            catch (JsonSerializationException ex)
            {
                Debug.LogError("Json Serialized Error: " + ex.Message);
                Debug.LogError("Stack Trace: " + ex.StackTrace);
                if (ex.InnerException != null)
                {
                    Debug.LogError("Inner Exception: " + ex.InnerException.Message);
                }
            }
            catch (Exception ex)
            {
                Debug.LogError("General Error: " + ex.Message);
                InfoText[1].text = "General Error: " + ex.Message;
            }
        }
    }
}
```
* 앞선 포스팅의 StartNavigate()에서 일부 수정되었다.
<br>
<br>
<br>
UpdateNavigation()

```c#
 IEnumerator UpdateNavigation()
 {
     while (true)
     {
         if (Input.location.status == LocationServiceStatus.Running)
         {
             float currentLat = Input.location.lastData.latitude;
             float currentLon = Input.location.lastData.longitude;

             DisplayCurrentLocationMessage(currentLat.ToString("F7"), currentLon.ToString("F7"));
         }
         yield return new WaitForSeconds(5.0f); // Update every 5 seconds
     }
 }
```
* while(true)와 yield return new WaitForSeconds(n)으로 길 안내 메세지 상태의를 주기별로 체크한다.

<br>
<br>
<br>

DisplayCurrentLocationMessage()

```c#
void DisplayCurrentLocationMessage(string latitude, string longitude)
{
    if (float.TryParse(latitude, NumberStyles.Float, CultureInfo.InvariantCulture, out float flt_latitude) &&
        float.TryParse(longitude, NumberStyles.Float, CultureInfo.InvariantCulture, out float flt_longitude))
    {
        InfoText[1].text = "flt_latitude: " + flt_latitude.ToString() + ", flt_longitude: " + flt_longitude.ToString();

        if (currentRoute != null && currentRoute.features != null)
        {
            float closestDistance = float.MaxValue;
            string closestDescription = null;

            foreach (var feature in currentRoute.features)
            {
                if (feature.geometry.coordinates != null && feature.geometry.coordinates.Count > 0)
                {
                    foreach (var coord in feature.geometry.coordinates)
                    {
                        if (coord is JArray coords && coords.Count >= 2)
                        {
                            float pointLat = Convert.ToSingle(coords[1], CultureInfo.InvariantCulture);
                            float pointLng = Convert.ToSingle(coords[0], CultureInfo.InvariantCulture);

                            float distance = GetDistance(flt_latitude, flt_longitude, pointLat, pointLng);
                            if (distance < closestDistance)
                            {
                                closestDistance = distance;
                                closestDescription = feature.properties.description;
                            }
                        }
                        else if (coord is List<object> list && list.Count >= 2)
                        {
                            float pointLat = Convert.ToSingle(list[1], CultureInfo.InvariantCulture);
                            float pointLng = Convert.ToSingle(list[0], CultureInfo.InvariantCulture);

                            float distance = GetDistance(flt_latitude, flt_longitude, pointLat, pointLng);
                            if (distance < closestDistance)
                            {
                                closestDistance = distance;
                                closestDescription = feature.properties.description;
                            }
                        }
                    }
                }
            }

            if (closestDistance < 0.01f) // threshold 설정
            {
                InfoText[1].text = closestDescription;
            }
            else
            {
                InfoText[1].text = "No nearby points found.";
            }
        }
        else
        {
            InfoText[1].text = "No route data available.";
        }
    }
    else
    {
        InfoText[0].text = "Failed to parse latitude or longitude.";
    }
}

private float GetDistance(float lat1, float lon1, float lat2, float lon2)
{
    return Mathf.Sqrt(Mathf.Pow(lat1 - lat2, 2) + Mathf.Pow(lon1 - lon2, 2));
}
```

* 현재 위치에 해당하는 길 안내 메세지를 띄우기 위해, Json 데이터에  <"순서","위경도","정보"> 형식으로 저장되어 있는 노드의 "위경도" 값을 현재 GPS의 위경도와 비교하고, 가장 근접한 곳의 "정보" 를 선택한다. 또한 GPS가 한번씩 튀는 것을 고려하여 임계값을 설정해준다. 
<br>
<br>
이후 비정상적인 작동은 디버그로그를 통해 확인하고 고칠 수 있다!
