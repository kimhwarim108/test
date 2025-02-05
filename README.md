import cv2
import gradio as gr
import requests
import numpy as np
from PIL import Image
from requests.auth import HTTPBasicAuth

# Vision AI API 설정
VISION_API_URL = "https://suite-endpoint-api-apne2.superb-ai.com/endpoints/081e92bb-0e78-421d-a403-60b385822772/inference"
TEAM = "kdt2025_1-27"
ACCESS_KEY = "qdZfd61CL52PqUOGTjmiea7KHDYIOvO55PsHDIss"

# 클래스별 색상 지정
def choose_color(name):
    color_map = {
        'RASPBERRY PICO': (0, 255, 0),
        'USB': (255, 0, 0),
        'HOLE': (0, 0, 255),
        'OSCILLATOR': (0, 0, 0),
        'CHIPSET': (0, 126, 126),
        'BOOTSEL': (255, 255, 255)
    }
    return color_map.get(name, (128, 128, 128))  # 기본값(회색)

# 바운딩 박스 그리기
def draw_pic(img, name, start1, start2, end1, end2):
    start_point = (start1, start2)
    end_point = (end1, end2)
    color = choose_color(name)
    thickness = 2
    cv2.rectangle(img, start_point, end_point, color, thickness)

# 객체 이름 표시
def draw_name(img, name, start1, start2):
    text = name
    position = (start1, start2 - 10)
    font = cv2.FONT_HERSHEY_SIMPLEX
    font_scale = 0.8
    color = choose_color(name)
    thickness = 2
    cv2.putText(img, text, position, font, font_scale, color, thickness, cv2.LINE_AA)

# API를 호출하고 바운딩 박스를 그리는 함수
def process_image(image):
    # 이미지를 OpenCV 형식으로 변환
    image = np.array(image)
    image = cv2.cvtColor(image, cv2.COLOR_RGB2BGR)

    # 이미지를 API에 전송할 수 있는 형식으로 변환
    _, img_encoded = cv2.imencode(".jpg", image)
    image_bytes = img_encoded.tobytes()

    # API 호출
    response = requests.post(
        url=VISION_API_URL,
        auth=HTTPBasicAuth(TEAM, ACCESS_KEY),
        headers={"Content-Type": "image/jpeg"},
        data=image_bytes,
    )

    # API 응답 처리
    result = response.json()
    objects = result.get("objects", [])

    # BBox 및 클래스 이름 표시
    name_dict = {}
    for obj in objects:
        names = obj["class"]
        box = obj["box"]
        draw_pic(image, names, box[0], box[1], box[2], box[3])
        draw_name(image, names, box[0], box[1])
        
        name_dict[names] = name_dict.get(names, 0) + 1

    # 탐지된 객체 개수 출력
    i = 50
    for k, v in name_dict.items():
        text = f"{k} : {v}"
        position = (50, i)
        font = cv2.FONT_HERSHEY_SIMPLEX
        font_scale = 0.8
        color = choose_color(k)
        thickness = 2
        cv2.putText(image, text, position, font, font_scale, color, thickness, cv2.LINE_AA)
        i += 30

    # OpenCV에서 RGB로 변환 후 반환
    image = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)
    return Image.fromarray(image)

# Gradio 인터페이스 설정
iface = gr.Interface(
    fn=process_image,
    inputs=gr.Image(type="pil"),
    outputs="image",
    title="Vision AI Object Detection",
    description="Upload an image to detect objects using Vision AI. Bounding boxes and class names will be displayed.",
)

# 인터페이스 실행
iface.launch()
