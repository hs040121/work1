<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <title>전국 학교 근처 도서관 검색</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
        }

        h1 {
            background-color: #4CAF50;
            color: white;
            padding: 20px;
            text-align: center;
            margin: 0;
        }

        form {
            display: flex;
            flex-wrap: wrap;
            justify-content: center;
            align-items: center;
            gap: 10px;
            margin: 20px 0;
        }

        select, input[type="text"], input[type="number"] {
            width: 250px;
            padding: 10px;
            border: 1px solid #ddd;
            border-radius: 5px;
            font-size: 16px;
        }

        button {
            padding: 10px 20px;
            font-size: 16px;
            color: white;
            background-color: #4CAF50;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        }

        button:hover {
            background-color: #45a049;
        }

        #map {
            width: 100%;
            height: 400px;
        }

        #resultsTable {
            width: 90%;
            margin: 20px auto;
            max-height: 300px;
            overflow-y: auto;
            border: 1px solid #ddd;
        }

        table {
            width: 100%;
            border-collapse: collapse;
        }

        th, td {
            border: 1px solid #ddd;
            text-align: left;
            padding: 12px;
        }

        th {
            background-color: #f4f4f4;
            text-align: center;
        }

        tr:hover {
            background-color: #f1f1f1;
            cursor: pointer;
        }
    </style>
    <script type="text/javascript" src="https://oapi.map.naver.com/openapi/v3/maps.js?ncpClientId=1igppkmtqd&submodules=geocoder"></script>
</head>
<body>
    <h1>전국 학교 근처 도서관 검색</h1>
    <form id="searchForm">
        <select id="regionSelect" required>
            <option value="">지역 선택</option>
            <option value="서울">서울</option>
            <option value="부산">부산</option>
            <option value="대구">대구</option>
            <option value="인천">인천</option>
            <option value="광주">광주</option>
            <option value="대전">대전</option>
            <option value="울산">울산</option>
            <option value="경기">경기</option>
            <option value="강원">강원</option>
            <option value="충북">충북</option>
            <option value="충남">충남</option>
            <option value="전북">전북</option>
            <option value="전남">전남</option>
            <option value="경북">경북</option>
            <option value="경남">경남</option>
            <option value="제주">제주</option>
        </select>
        <select id="schoolTypeSelect" required>
            <option value="">학교 구분 선택</option>
            <option value="초등학교">초등학교</option>
            <option value="중학교">중학교</option>
            <option value="고등학교">고등학교</option>
        </select>
        <input type="text" id="queryInput" placeholder="학교 이름" required>
        <input type="number" id="distanceInput" placeholder="거리(Km)" required>
        <button type="submit">검색</button>
    </form>

    <div id="map"></div>
    <div id="resultsTable"></div>

    <script>
        const kcisaApiUrl = "http://api.kcisa.kr/openapi/API_CNV_065/request";
        const kcisaServiceKey = "b4ff56c8-b1eb-4fd5-a14f-b934d62cef85";

        let map;
        let markers = [];
        let highlightedMarker = null; // 현재 하이라이트된 마커

        function initMap() {
            map = new naver.maps.Map('map', {
                center: new naver.maps.LatLng(37.3595704, 127.105399),
                zoom: 10
            });
        }

        function addMarker(lat, lng, title, color, infoContent) {
            const marker = new naver.maps.Marker({
                position: new naver.maps.LatLng(lat, lng),
                map: map,
                title: title,
                icon: {
                    url: `https://maps.google.com/mapfiles/ms/icons/${color}-dot.png`, // Google Maps 스타일 핀
                    size: new naver.maps.Size(32, 32), // 핀 크기
                    anchor: new naver.maps.Point(16, 32), // 핀 앵커 지점 설정
                }
            });

            const infoWindow = new naver.maps.InfoWindow({
                content: `<div style="padding:10px;"><strong>${title}</strong><br>${infoContent}</div>`
            });

            naver.maps.Event.addListener(marker, 'click', function () {
                infoWindow.open(map, marker);
            });

            markers.push(marker);

            return marker;
        }

        function clearMarkers() {
            markers.forEach(marker => marker.setMap(null));
            markers = [];
            if (highlightedMarker) {
                highlightedMarker.setIcon({
                    url: `https://maps.google.com/mapfiles/ms/icons/blue-dot.png`, // 파란색 핀으로 복원
                    size: new naver.maps.Size(32, 32),
                    anchor: new naver.maps.Point(16, 32),
                });
                highlightedMarker = null;
            }
        }

        function highlightMarker(marker) {
            if (highlightedMarker) {
                highlightedMarker.setIcon({
                    url: `https://maps.google.com/mapfiles/ms/icons/blue-dot.png`, // 파란색 핀으로 복원
                    size: new naver.maps.Size(32, 32),
                    anchor: new naver.maps.Point(16, 32),
                });
            }

            marker.setIcon({
                url: `https://maps.google.com/mapfiles/ms/icons/yellow-dot.png`, // 노란색 핀으로 변경
                size: new naver.maps.Size(32, 32),
                anchor: new naver.maps.Point(16, 32),
            });

            highlightedMarker = marker;
        }

        document.getElementById('searchForm').addEventListener('submit', function (event) {
            event.preventDefault();

            const region = document.getElementById('regionSelect').value;
            const schoolType = document.getElementById('schoolTypeSelect').value;
            const query = document.getElementById('queryInput').value;
            const distance = document.getElementById('distanceInput').value;
            const resultsContainer = document.getElementById('resultsTable');

            if (!region || !schoolType) {
                alert("지역과 학교 구분을 선택하세요.");
                return;
            }

            resultsContainer.innerHTML = "";

            const url = `${kcisaApiUrl}?serviceKey=${kcisaServiceKey}&numOfRows=100&pageNo=1&schNm=${encodeURIComponent(query)}&dist=${distance}&type=xml`;

            fetch(url)
                .then(response => response.text())
                .then(xmlText => {
                    const parser = new DOMParser();
                    const xmlDoc = parser.parseFromString(xmlText, "text/xml");
                    const items = xmlDoc.getElementsByTagName("item");

                    clearMarkers();

                    if (items.length > 0) {
                        const table = document.createElement("table");
                        const thead = document.createElement("thead");
                        const headerRow = document.createElement("tr");
                        ["학교", "도서관", "주소", "운영일", "열람좌석"].forEach(header => {
                            const th = document.createElement("th");
                            th.textContent = header;
                            headerRow.appendChild(th);
                        });
                        thead.appendChild(headerRow);
                        table.appendChild(thead);

                        const tbody = document.createElement("tbody");

                        Array.from(items).forEach(item => {
                            const schoolName = item.getElementsByTagName("schNm")[0]?.textContent || "정보 없음";
                            const libraryName = item.getElementsByTagName("fcltyNm")[0]?.textContent || "정보 없음";
                            const address = item.getElementsByTagName("fcltyRoadNmAddr")[0]?.textContent || "정보 없음";
                            const description = item.getElementsByTagName("description")[0]?.textContent || "정보 없음";
                            const subDescription = item.getElementsByTagName("subDescription")[0]?.textContent || "정보 없음";

                            const lat = parseFloat(item.getElementsByTagName("schLatPos")[0]?.textContent || "0");
                            const lng = parseFloat(item.getElementsByTagName("schLotPos")[0]?.textContent || "0");

                            if (!address.includes(region)) return;
                            if (!schoolName.includes(schoolType)) return;

                            const row = document.createElement("tr");
                            [schoolName, libraryName, address, description, subDescription].forEach(cellData => {
                                const td = document.createElement("td");
                                td.textContent = cellData;
                                row.appendChild(td);
                            });
                            tbody.appendChild(row);

                            let schoolMarker = null;
                            let libraryMarker = null;

                            if (!isNaN(lat) && !isNaN(lng)) {
                                schoolMarker = addMarker(lat, lng, schoolName, "red", address); // 학교는 빨간색 핀
                            }

                            row.addEventListener('click', () => {
                                if (schoolMarker) {
                                    map.panTo(schoolMarker.getPosition());
                                    highlightMarker(schoolMarker);
                                }
                            });

                            naver.maps.Service.geocode({ query: address }, (status, response) => {
                                if (status === naver.maps.Service.Status.OK) {
                                    const result = response.v2.addresses[0];
                                    const libraryLat = parseFloat(result.y);
                                    const libraryLng = parseFloat(result.x);
                                    libraryMarker = addMarker(libraryLat, libraryLng, libraryName, "blue", address); // 도서관은 파란색 핀

                                    row.addEventListener('click', () => {
                                        map.panTo(libraryMarker.getPosition());
                                        highlightMarker(libraryMarker);
                                    });
                                }
                            });
                        });

                        table.appendChild(tbody);
                        resultsContainer.appendChild(table);
                    } else {
                        resultsContainer.innerHTML = "<p>검색 결과가 없습니다.</p>";
                    }
                })
                .catch(error => console.error("오류 발생:", error));
        });

        initMap();
    </script>
</body>
</html>
