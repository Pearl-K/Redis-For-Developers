# Redis Geospatial Index
위치 데이터는 주로 경도, 위도의 좌표 쌍으로 표현되는데 이런 공간 데이터를 사용해서 다양한 기능을 만들어야할 때가 있다.


- 사용자의 현재 위치 파악
- 사용자 이동에 따른 실시간 변동 위치 업데이트
- 사용자 위치 기준으로 근처 장소 검색


만약, 사용자가 자주 이동하여 위치 데이터 write가 반복적으로 일어난다면 DB 계층에서 처리하기 까다로운 문제가 된다.


## Redis의 위치 데이터
- geo 자료구조를 통해 공간 정보 데이터 처리
- Redis가 인메모리 DB라는 장점에 힘입어 쓰기 연산 or 공간 데이터를 활용한 연산이 빠르게 처리될 수 있다.


### geo set
- 위치 공간 관리에 특화된 데이터 구조
- (경도, 위도) 쌍으로 데이터 저장
- 내부적으로 Sorted Set 구조로 저장된다.


### 대표적인 커맨드
- `GEOADD`: 위도/경도 좌표를 member 이름으로 저장 (ZSET 내부에 geohash로 저장)
- `GEOPOS`: 저장된 멤버들의 현재 위도/경도 좌표 반환
- `GEOSEARCH`: (아래)옵션을 주고 찾는 커맨드


### 추가 옵션들
- `BYRADIUS`: 원형 영역 기준 검색 (m, km, ft, mi 지원)
- `BYBOX`: 직사각형 영역 기준 검색 (width, height, 단위 지정 필요)


### 사용 흐름 (예시 코드)
```bash
# 1. 위치 데이터 저장
# GEOADD key 경도 위도 멤버
GEOADD cafe-locations 127.0276 37.4979 "Gangnam Cafe"
GEOADD cafe-locations 126.9780 37.5665 "City Hall Cafe"
GEOADD cafe-locations 126.9900 37.5600 "Myeongdong Cafe"

# 2. 특정 멤버의 좌표 조회
GEOPOS cafe-locations "Gangnam Cafe"

# 3. 반경 검색 (반경 5km 이내 카페 찾기)
GEOSEARCH cafe-locations FROMMEMBER "Gangnam Cafe" BYRADIUS 5 km ASC COUNT 5

# 4. 박스 검색 (가로 10km × 세로 5km 박스 내 카페 찾기)
GEOSEARCH cafe-locations FROMMEMBER "City Hall Cafe" BYBOX 10 5 km ASC

# 5. 좌표 기준 검색 (경도, 위도 직접 지정)
GEOSEARCH cafe-locations FROMLONLAT 127.0300 37.5000 BYRADIUS 2 km

# 6. 거리 계산 (두 지점 간 거리) → km 단위 거리 반환
GEODIST cafe-locations "Gangnam Cafe" "City Hall Cafe" km
```