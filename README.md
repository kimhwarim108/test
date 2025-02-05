import cv2
import os
import requests
from requests.auth import HTTPBasicAuth
from collections import Counter
# :열린_파일_폴더: 폴더 경로 설정
input_folder = "/home/woosbus/vision_ai_ws/data_set/"
output_folder = "/home/woosbus/vision_ai_ws/data_set/output/"
# :링크: API 정보
URL = "https://suite-endpoint-api-apne2.superb-ai.com/endpoints/081e92bb-0e78-421d-a403-60b385822772/inference"
ACCESS_KEY = "qdZfd61CL52PqUOGTjmiea7KHDYIOvO55PsHDIss"
# :예술: 객체별 색상 지정
color_map = {
    "HOLE": (0, 255, 0),  # 초록색
    "USB": (0, 0, 255),  # 빨간색
    "OSCILLATOR": (255, 0, 0),  # 파란색
    "RASPBERRY_PICO": (255, 255, 0),  # 진한 초록색
    "CHIPSET": (255, 150, 255),  # 다른 색상
    "BOTSEL": (30, 100, 255)  # 추가 색상
}
# :액자에_담긴_그림: 입력 폴더 내 모든 JPG 파일 리스트
image_files = [f for f in os.listdir(input_folder) if f.endswith(".jpg")]
# :열린_파일_폴더: 출력 폴더 생성 (없으면 생성)
os.makedirs(output_folder, exist_ok=True)
# :끝이_왼쪽_아래를_향한_붓: 폴더 내 이미지 하나씩 처리
for image_file in image_files:
    img_path = os.path.join(input_folder, image_file)  # 원본 이미지 경로
    output_path = os.path.join(output_folder, image_file)  # 저장할 이미지 경로
    # :흰색_확인_표시: 1. 이미지 불러오기
    img = cv2.imread(img_path)
    if img is None:
        print(f":x: 오류: {img_path} 이미지를 불러올 수 없습니다.")
        continue
    # :흰색_확인_표시: 2. 객체 탐지 API 호출
    with open(img_path, "rb") as image:
        response = requests.post(
            url=URL,
            auth=HTTPBasicAuth("kdt2025_1-27", ACCESS_KEY),
            headers={"Content-Type": "image/jpeg"},
            data=image.read(),
        )
    if response.status_code != 200:
        print(f":x: API 요청 실패: {img_path}")
        continue
    result = response.json()
    objects = result.get("objects", [])  # 객체 정보 추출
    # :흰색_확인_표시: 3. 객체 Dictionary 생성
    my_dict = {}
    for obj in objects:
        obj_class = obj["class"]
        box = obj["box"]
        my_dict[obj_class] = box  # 객체와 바운딩 박스 저장
    # :흰색_확인_표시: 4. 객체 개수 세기
    object_counts = Counter()
    for key in my_dict.keys():
        base_name = ''.join([c for c in key if not c.isdigit()])
        object_counts[base_name] += 1
    # :흰색_확인_표시: 5. 박스 그리기
    font = cv2.FONT_HERSHEY_SIMPLEX
    font_scale = 0.5
    thickness = 1
    for key, coords in my_dict.items():
        start_point = (coords[0], coords[1])
        end_point = (coords[2], coords[3])
        # 객체 색상 지정
        color = (0, 150, 30)  # 기본 색상
        for obj_name in color_map.keys():
            if obj_name in key:
                color = color_map[obj_name]
                break
        # 바운딩 박스 그리기
        cv2.rectangle(img, start_point, end_point, color, 2)
        # 객체 이름 표시 (바운딩 박스 위)
        text_position = (start_point[0], start_point[1] - 5)
        cv2.putText(img, key, text_position, font, font_scale, color, thickness, cv2.LINE_AA)
    # :흰색_확인_표시: 6. 좌상단에 객체 정보 출력
    text_color = (255, 0, 255)
    y_offset = 20
    for idx, (obj, count) in enumerate(object_counts.items()):
        text = f"{obj}: {count}"
        position = (10, 30 + idx * y_offset)
        cv2.putText(img, text, position, font, font_scale, text_color, thickness, cv2.LINE_AA)
    # :흰색_확인_표시: 7. 이미지 저장
    cv2.imwrite(output_path, img)
    print(f":흰색_확인_표시: 저장 완료: {output_path}")
print(":짠: 모든 이미지 처리가 완료되었습니다!")
