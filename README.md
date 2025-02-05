import requests
from requests.auth import HTTPBasicAuth
import cv2
import os

def choose_color(name):
    """클래스 이름별 BBox 색상 선택"""
    class_colors = {
        'RASPBERRY PICO': (0, 255, 0),           # 초록색
        'USB': (255, 0, 0),            # 파란색
        'HOLE': (0, 0, 255),           # 빨간색
        'OSCILLATOR': (255, 255, 0),   # 하늘색
        'CHIPSET': (255, 0, 255),      # 보라색
        'BOOTSEL': (255, 255, 255)     # 흰색
    }
    return class_colors.get(name, (128, 128, 128))  # 기본값 (회색)

def draw_pic(img, name, start1, start2, end1, end2):
    """Box 그리기"""
    start_point = (start1, start2)
    end_point = (end1, end2)
    color = choose_color(name)
    thickness = 2
    cv2.rectangle(img, start_point, end_point, color, thickness)

def draw_name(img, name, start1, start2):
    """이름 표시"""
    text = name
    position = (start1, start2 - 10)  # 텍스트 위치 (BBox 위)
    font = cv2.FONT_HERSHEY_SIMPLEX
    font_scale = 0.8
    color = choose_color(name)
    thickness = 2
    cv2.putText(img, text, position, font, font_scale, color, thickness, cv2.LINE_AA)

# API 설정
URL = "https://suite-endpoint-api-apne2.superb-ai.com/endpoints/081e92bb-0e78-421d-a403-60b385822772/inference"
ACCESS_KEY = "qdZfd61CL52PqUOGTjmiea7KHDYIOvO55PsHDIss"

# 입력 및 저장 폴더 설정
input_folder = "picture"  # 원본 이미지 폴더
output_folder = "save"    # 저장할 폴더

# 입력 폴더 및 저장 폴더 생성
os.makedirs(input_folder, exist_ok=True)  # 입력 폴더가 없으면 생성
os.makedirs(output_folder, exist_ok=True) # 저장 폴더가 없으면 생성

# 이미지 처리
image_files = sorted(os.listdir(input_folder))  # 파일 리스트 정렬
if not image_files:
    print("입력 폴더에 이미지 파일이 없습니다.")
    exit()

for image_file in image_files:
    img_path = os.path.join(input_folder, image_file)

    # 이미지 로드
    if not img_path.lower().endswith(('.jpg', '.jpeg', '.png')):
        continue  # 이미지 파일이 아니면 건너뛰기
    image = open(img_path, "rb").read()
    img = cv2.imread(img_path)
    if img is None:
        print(f":x: 이미지를 불러올 수 없음: {img_path}")
        continue

    # API 요청
    response = requests.post(
        url=URL,
        auth=HTTPBasicAuth("kdt2025_1-27", ACCESS_KEY),
        headers={"Content-Type": "image/jpeg"},
        data=image,
    )

    # API 응답 확인
    try:
        result = response.json()
        objects = result["objects"]
    except Exception as e:
        print(f":x: JSON 파싱 오류: {img_path} ({e})")
        continue

    # BBox 및 클래스 이름 표시
    name_dict = {}
    for obj in objects:
        names = obj["class"]
        box = obj["box"]
        draw_pic(img, names, box[0], box[1], box[2], box[3])
        draw_name(img, names, box[0], box[1])
        name_dict[names] = name_dict.get(names, 0) + 1  # 개수 카운트

    # 클래스별 개수 출력
    i = 50
    for k, v in name_dict.items():
        text = f"{k} : {v}"
        position = (50, i)
        font = cv2.FONT_HERSHEY_SIMPLEX
        font_scale = 0.8
        color = choose_color(k)
        thickness = 2
        cv2.putText(img, text, position, font, font_scale, color, thickness, cv2.LINE_AA)
        i += 30
        print(text)

    # 결과 이미지 저장
    output_path = os.path.join(output_folder, image_file)
    cv2.imwrite(output_path, img)
