### 1. 문제 정의

카메라를 이용한 객체 인식 기술은 다양한 분야(보안, 교통, 산업 자동화 등)에서 핵심 기술로 자리잡고 있다. 특히, **모바일 환경에서 실시간 객체 인식 기능을 웹 애플리케이션 형태로 제공하는 서비스**는 사용자 접근성과 확장성 면에서 매우 유용하다. 본 프로젝트는 YOLOv8 모델을 활용하여 웹 브라우저 기반으로 실시간 객체 인식 기능을 구현하고, 스마트폰에서도 사용 가능하도록 하는 것을 목표로 한다.

---

### 2. 요구사항 분석

| 구분 | 세부 항목 |
| --- | --- |
| 입력 | 스마트폰 카메라 또는 노트북 카메라 |
| 처리 | YOLOv8을 이용한 객체 감지 및 바운딩 박스 시각화 |
| 출력 | 감지된 이미지와 객체 목록(JSON) |
| 인터페이스 | FastAPI 백엔드 + React(JavaScript) 프론트엔드 |
| 사용성 | 모바일 브라우저에서 접속 가능(ngrok HTTPS 사용) |
| 기능 | 후면/전면 카메라 전환, 자동 주기 감지, 수동 감지, 이미지 전송 및 결과 출력 |

---

### 3. 기술 스택 및 시스템 아키텍처

| 영역 | 사용 기술 |
| --- | --- |
| 인공지능 모델 | YOLOv8 (ultralytics) |
| 백엔드 | FastAPI, Uvicorn |
| 프론트엔드 | React (CDN 기반), JavaScript |
| 이미지 처리 | OpenCV, Pillow, NumPy |
| 배포 환경 | Google Colab + pyngrok |
| 통신 방식 | REST API (이미지 업로드 및 JSON 응답) |

### 📌 시스템 아키텍처

```
[Mobile Browser]
     ⬇ 이미지 전송
[ngrok Public URL]
     ⬇ POST /api/detect
[FastAPI on Colab]
     ⬇ YOLOv8로 이미지 분석
[객체 감지 결과 반환]
     ⬆ 바운딩 이미지 + JSON

```

---

### 4. 핵심 알고리즘 및 처리 메커니즘

- **YOLOv8 객체 인식**
    - OpenCV로 업로드된 이미지를 NumPy 배열로 디코딩
    - YOLOv8의 `.predict()` 메서드로 객체 탐지
    - 클래스 이름, 신뢰도, 바운딩 박스 좌표 추출
- **이미지 후처리 및 응답**
    - 감지된 객체에 대한 사각형 및 라벨 표시
    - 최종 이미지를 base64 인코딩 후 JSON으로 반환
    - 프론트엔드에서 `<img src="data:image/jpeg;base64,...">`로 표시
- **모바일 카메라 연동**
    - WebRTC 기반 `getUserMedia()`를 통해 후면/전면 카메라 제어
    - 비디오 프레임 캡처 → canvas로 이미지 캡처 → Blob 전송

---

### 5. 구현 과정 및 주요 코드

### 📌 FastAPI 서버 (`app.py`)

```python
@app.post("/api/detect")
async def detect_objects(image: UploadFile = File(...)):
    contents = await image.read()
    img = cv2.imdecode(np.frombuffer(contents, np.uint8), cv2.IMREAD_COLOR)
    results = model(img)[0]

    for box in results.boxes:
        x1, y1, x2, y2 = map(int, box.xyxy[0])
        # 바운딩 박스 및 텍스트 추가 (OpenCV)

```

### 📌 React 기반 프론트엔드 (`app.js`)

```jsx
const detectObjects = async () => {
    // canvas에서 프레임 추출 → blob 생성
    const blob = await new Promise(resolve => canvas.toBlob(resolve, 'image/jpeg'));
    const formData = new FormData();
    formData.append('image', blob, 'capture.jpg');

    const res = await fetch('/api/detect', { method: 'POST', body: formData });
    const data = await res.json();
    setDetectedObjects(data.detected_objects);
    setProcessedImage(data.processed_image);
};

```

---

### 6. 문제 해결 및 디버깅 경험

| 문제 | 해결 방법 |
| --- | --- |
| FileNotFoundError | `www/templates`, `www/static/js` 등 디렉토리 사전 생성 필수 |
| CORS 오류 | FastAPI에 `CORSMiddleware` 추가하여 모든 Origin 허용 |
| 500 Internal Server Error | YOLO 모델 미설치, 이미지 디코딩 오류 → 모델 설치 확인 및 에러 로그 분석 |
| 카메라 미작동 | HTTPS 접속 필수 (ngrok 사용), 모바일 권한 설정 확인 |
| 속도 문제 | `yolov8n.pt`와 같이 작은 모델 사용, 감지 주기 증가(10초)로 완화 |

---

### 7. 습득한 기술 및 인사이트

- YOLOv8 모델을 FastAPI에 통합하여 실시간 객체 감지 API 구현 경험
- 모바일 브라우저에서 카메라 연동을 위한 `MediaDevices.getUserMedia` 활용
- 캔버스를 이용한 실시간 프레임 캡처 및 이미지 업로드 처리
- base64 인코딩된 이미지를 HTML에 직접 출력하는 방식 숙지
- Colab 환경에서 pyngrok을 통한 외부 배포 및 서버 유지 방법 익힘

---

### 8. 결론 및 향후 개선 방향

**결론**

YOLOv8과 FastAPI, React를 통합하여 브라우저 기반 실시간 객체 인식 시스템을 성공적으로 구현했다. Colab 환경만으로도 복잡한 AI 서비스가 가능한 구조를 실습하며 모델 통합, 웹서버 운영, UI 구성 등 전반적인 웹 서비스 개발 과정을 체험할 수 있었다.

**개선 방향**

- 감지된 객체 필터링 (예: `person`, `car`만 표시)
- 객체 감지 결과를 서버 측 로그로 저장
- TensorRT로 모델 최적화 또는 YOLOv8-light 적용
- 전처리 이미지 리사이징 자동화
- 감지 주기 설정 사용자화 (ex: 슬라이더 UI 추가)

---

### 9. 온라인 포트폴리오 링크

GitHub : https://github.com/gunGeongun/yolo-webapp

### 10. 실제 시연

![image](https://github.com/user-attachments/assets/a7c50b2f-f673-473b-8b4d-2f0292002385)
