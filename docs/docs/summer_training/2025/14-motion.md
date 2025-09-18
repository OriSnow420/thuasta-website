# 香橙派和智能小车

## 前置：Copilot必备！

- [常见登不上去的情况](https://zhuanlan.zhihu.com/p/1908234855482889076)

- 【安装网络问题】pip install opencv-python --trusted-host pypi.tuna.tsinghua.edu.cn --trusted-host files.pythonhosted.org --trusted-host pypi.org

- 建议重新烧录一下系统，相关资料上传在云盘：[云盘连接](https://cloud.tsinghua.edu.cn/d/a3c7a2ad2ba04069a30e/)

## 主要内容

- 准备工作
  - ```git clone https://github.com/thuasta/thuei-1.git```克隆代码仓库
  - 创建新的conda环境```conda create -n masterpi --clone base```克隆基础环境
  - 程序中```sys.path.append('/home/pi/MasterPi/')```改成```sys.path.append('/root/thuei-1/sdk-python/')```即主文件夹所在路径（之后别的程序也都要改，可以考虑查找替换一劳永逸）
  - ColorDetect.py中```from ai_pro_pin_map import _BOARD```原先大小写打错了
  - 开i2c修改权限（可以直接用root干所以事情，虽然方便，但这个习惯可能不大好）
    - ```udevadm info -a /dev/i2c-7```查看信息：```KERNEL=="i2c-7"``````SUBSYSTEM=="i2c-dev"```
    - ```sudo touch /etc/udev/rules.d/50-myi2c.rules```
    - ```sudo vim /etc/udev/rules.d/50-myi2c.rules```
    - 按I进入插入模式，粘贴```KERNEL=="i2c-7", SUBSYSTEM=="i2c-dev", GROUP="users", MODE="0666"```
    - 按```ESC```->```:wq```保存退出
    - ```sudo udevadm control --reload```
    - 重启设备即可启用

结合源码学习（MasterPi原始文件夹下）

- 麦轮小车运动学
  - Board::setMotor():
    - with语句：try...except...finally的浓缩写法，自动处理资源释放
  - I2C：```with SMBus(__i2c) as bus:```使用SMBus类创建I2C总线对象，__i2c参数指定总线编号
  - 如何查看I2C编号：```i2cdetect -l```命令列出所有I2C总线(记得打开电源)
  - 如何查看地址：```sudo i2cdetect -y -a -r 7```命令扫描I2C总线上所有设备的地址，-y表示不提示确认，-a表示显示所有地址（包括保留地址），-r表示使用快速模式；注意树莓派默认是1，香橙派默认是7
  - 常见问题：``` [Errno 5] Input/output error```请检查I2C连接、电池电量
  - 尝试运行```MotorControlDemo.py```
    - 知识点：使用signal模块捕获SIGINT信号​
  - 尝试运行```thuei-2/MecanumControl/Car_Forward_Demo.py```确认四个轮子的编号：
    - classis那行语句改成```chassis = mecanum.MecanumChassis(wheel_init_dir=[1, 1, 1, 1], wheel_init_map=[1, 2, 3, 4])```
    - 参考编号在```thuei-2/HiwonderSDK/mecanum.py```：
      - __init__函数头改成```def __init__(self, a=67, b=59, wheel_diameter=65, wheel_init_dir=[1,1,1,1], wheel_init_map=[1,2,3,4]):```；且加上```Board.initMotors(motor_init_dir=wheel_init_dir, motor_init_map=wheel_init_map)```
    - 此时运行四个轮子可能是瞎转的，但鉴于插拔实在有点费力，考虑在软件上解决这个问题
    - 记得改完后作为MecanumChassis的默认值
  - 结合pdf了解麦轮的基本用法

- opencv图像处理初步——颜色识别
  - 从这里开始，因为需要图形化界面，建议用vnc，mobaxterm也行
  - TightVNC设置密码：```vncpasswd```命令设置VNC连接密码（首次使用需要设置）
  - 启动VNC服务器：```vncserver :1 -geometry 1920x1080```设置成1080P
  - 主要了解颜色空间转换
  - 运行python3 Functions/ColorDetect.py，注意会有报错```module 'HiwonderSDK.Board' has no attribute 'RGB'```，要把.py文件中RGB相关的代码删掉，我们用不到
  - ```cap = cv2.VideoCapture('http://127.0.0.1:8080?action=stream')```括号里改成0，摄像头USB插到香橙派上
  - 摄像头相关错误解决：
    - ```VIDEOIO(V4L2:/dev/video0): can't open camera```表示无法打开摄像头
    - 因为没连USB
  - 如果遇到```No module named OPi```报错，就```pip install --upgrade OPi.GPIO```即可
- 涉及蜂鸣器相关的参见技术文档
  - 记得连31号端口

## 后续自主探索

  - 循迹原理
    - 尝试阅读并执行```thuei-1/sdk-python/Functions/VisualPatrol.py```
    - 代码结构分析：
      - 核心思路：通过摄像头识别线条，计算偏移量，使用PID控制调整电机速度
      - 主要模块：
        1. 视觉处理模块：颜色识别、ROI区域分割、轮廓检测
        2. 控制模块：PID控制器、电机驱动
        3. 多线程模块：图像处理线程、运动控制线程
    - 关键技术点深入：
      - ROI分层处理：将图像分为上中下三部分，不同权重(0.1, 0.3, 0.6)，远处影响小，近处影响大
      - PID控制原理：P(比例)控制当前偏差，I(积分)消除稳态误差，D(微分)预测未来趋势
      - 颜色空间转换：BGR→LAB，LAB色彩空间对光照变化更稳定
      - 形态学处理：腐蚀去噪点→膨胀连接断线
    - 调试技巧：
      - 调整颜色阈值：修改yaml文件中的LAB范围
      - 调整PID参数：P值影响响应速度，I值影响稳态精度，D值影响稳定性
      - 观察中心点：通过cv2.circle显示检测到的线条中心
    - 知识点：PID控制、threading模块、OpenCV图像处理、ROI感兴趣区域
  - 避障实践
    - 尝试使用```Sonar.py```，有条件的可以用if-else实现横向避障
