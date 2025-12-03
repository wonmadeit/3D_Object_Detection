3D Object Detection ROS 2 Inference Node (CenterPoint + TensorRT)

본 패키지는 ROS 2 Foxy 환경에서 실시간 LiDAR PointCloud2 데이터를 입력받아,
TensorRT로 최적화된 CenterPoint 엔진을 사용하여 3D 객체 감지 추론을 수행한 후,
감지된 장애물의 2D 중심 좌표를 /imlab/obstacle 토픽으로 발행하는 ROS 노드입니다.

이 노드는 자율주행, 이동 로봇의 물체 감지 및 회피 시스템 등에 사용할 수 있도록 설계되었습니다.

1. 필수 환경 (Prerequisites)

노드가 정상적으로 실행되기 위한 환경은 아래와 같습니다.

구성 요소	요구 사항	비고
운영 체제 (OS)	Ubuntu 20.04	ROS 2 Foxy 공식 지원
ROS 2 버전	Foxy Fitzroy	apt 설치 가능
NVIDIA 드라이버	CUDA 11.7과 호환되는 버전	nvidia-smi로 버전 확인
CUDA Toolkit	11.7	모델 엔진 생성 환경과 동일해야 함
TensorRT	8.5.3.1	end2end.engine 생성 시 사용한 버전과 동일해야 함
Python 환경	Python 3.8 + ROS 2 Foxy	Conda 비활성화 필수
2. 설치 및 환경 설정
2.1. 파일 구조 확인

다음과 같은 폴더 구조를 가지고 있어야 합니다.
(스크립트 내부 경로는 상대 경로 기준으로 작성된 것으로 가정)

mmdeploy/ROS/3dod_ros2/
├── configs/
│   ├── deploy/
│   │   └── voxel-detection_tensorrt_dynamic-nus-20x5.py
│   └── model/
│       └── centerpoint_pillar02_second_secfpn_8xb4-cyclic-20e_kitti-3d.py
├── end2end.engine               # TensorRT 엔진 파일
├── inference_node_ros2.py       # ROS 2 추론 노드
└── requirements.txt

2.2. Python / ROS 환경 설정
1) Conda 비활성화 (매우 중요)

ROS 2는 Conda 환경과 충돌하므로 반드시 비활성화해야 합니다.

conda deactivate

2) ROS 2 Foxy 환경 설정
source /opt/ros/foxy/setup.bash

3) ros_test 가상환경 활성화
source /path/to/ros_test/bin/activate

4) 필요한 라이브러리 설치
pip install -r requirements.txt

3. 노드 실행 방법
3.1. 환경 변수 설정 (필수)

TensorRT 엔진 및 CUDA 라이브러리가 올바르게 로드되도록 환경 변수를 설정해야 합니다.
아래 경로는 시스템에 맞게 수정해야 합니다.

<예시>
export LD_LIBRARY_PATH="/usr/local/cuda-11.7/lib64:/home/gawon/Downloads/TensorRT-8.5.3.1/targets/x86_64-linux-gnu/lib:${LD_LIBRARY_PATH}"
unset NVIDIA_TF32_OVERRIDE

3.2. ROS 2 추론 노드 실행

mmdeploy/ROS/3dod_ros2 폴더로 이동하여 실행합니다.

python inference_node_ros2.py


노드는 /iv_points 토픽을 구독하기 전까지 대기 상태로 유지됩니다.

3.3. LiDAR 데이터 재생 (별도 터미널)
ros2 bag play 1_merged.db3


이때 bag 파일이 /iv_points 토픽을 포함해야 합니다.

4. ROS 2 토픽 정보
입력 (Subscription)
토픽 이름	메시지 타입	설명
/iv_points	sensor_msgs/msg/PointCloud2	LiDAR 포인트 클라우드 데이터
출력 (Publishing)
토픽 이름	메시지 타입	설명
/imlab/obstacle	std_msgs/msg/Float64MultiArray	감지된 장애물의 2D 중심 좌표 (최대 3개)
데이터 구조:
[x1, y1, x2, y2, x3, y3]


객체가 3개 미만일 경우 나머지는 0.0으로 패딩

각 박스의 중심은 BEV 기준 (LiDAR 좌표계 X-Y)

예:

[12.3, -1.8, 25.1, 0.4, 0.0, 0.0]

5. 동작 개요

/iv_points로 들어오는 LiDAR PointCloud2 데이터를 수신

포인트 데이터를 CenterPoint 모델 입력 구조(5D: X,Y,Z,Intensity,Ring)로 변환

TensorRT 엔진(end2end.engine)을 통해 실시간 추론 수행

감지된 객체의 3D bounding box 중 중심 좌표(X,Y) 를 추출

최대 3개의 객체 중심을 /imlab/obstacle로 발행

6. Known Issues & Notes

Conda 환경과 ROS 2 Foxy의 Python 패키지는 충돌할 수 있음 → 반드시 비활성화해야 함

TensorRT 버전이 엔진 파일 제작 시 사용한 버전과 다르면 로딩 불가

bag 파일의 토픽 이름이 /iv_points가 아닐 경우 inference 노드가 데이터를 수신하지 못함
