# 嵌入式系统常见信号处理与滤波算法总结

## 一、限幅滤波法

### 原理
设定最大允许偏差值(`MAX_DELTA`)，若新采样值与历史值的差值超过阈值则视为噪声，保留旧值。
### 应用场景
- 🛠️ 机械按键消抖
- ⚡ 电机转速突变检测
### 代码实现
```c
#define MAX_DELTA 5  // 最大允许偏差值

char limit_filter(char current_val) {
    static char last_val;
    if(abs(current_val - last_val) > MAX_DELTA) 
        return last_val;   // 超出阈值保留旧值
    else 
        last_val = current_val;
    return current_val;
}
```

## 二、中位值滤波

### 原理

对奇数次采样值排序后取中间值，抑制偶发干扰。

### 应用场景

- 🌡️ 热敏电阻测温去噪
- 💧 液位监测过滤流体波动

### 代码实现

```
#define N 7  // 奇数采样次数

char median_filter() {
    char samples[N], temp;
    for(int i=0; i<N; i++)
        samples[i] = get_adc();  // 获取采样值
        
    // 冒泡排序
    for(int i=0; i<N-1; i++) {
        for(int j=0; j<N-i-1; j++) {
            if(samples[j] > samples[j+1]) {
                temp = samples[j];
                samples[j] = samples[j+1];
                samples[j+1] = temp;
            }
        }
    }
    return samples[N/2];  // 返回中间值
}
```

## 三、滑动平均滤波

### 原理

维护固定长度数据队列，计算窗口内算术平均。

### 应用场景

- 📊 流量计数据平滑
- 🔋 电池电流监测

### 代码实现



```
#define WINDOW_SIZE 8  // 窗口长度

int moving_average() {
    static int buffer[WINDOW_SIZE], index=0;
    buffer[index] = get_adc();        // 新数据入队
    index = (index+1) % WINDOW_SIZE;  // 环形队列管理
    
    int sum = 0;
    for(int i=0; i<WINDOW_SIZE; i++)
        sum += buffer[i];
        
    return sum / WINDOW_SIZE;  // 计算均值
}
```

## 四、卡尔曼滤波

### 原理

通过预测-校正机制融合多传感器数据，动态估计系统状态。

### 应用场景

- 🚁 无人机姿态解算（融合陀螺仪/加速度计）
- 🚗 智能车定位（编码器+视觉数据）

### 代码实现



```
typedef struct {
    float q;  // 过程噪声方差
    float r;  // 测量噪声方差
    float x;  // 状态估计值
    float p;  // 估计误差协方差
    float k;  // 卡尔曼增益
} KalmanFilter;

float kalman_update(KalmanFilter* kf, float measurement) {
    // 预测阶段
    kf->p += kf->q;  
    
    // 更新阶段
    kf->k = kf->p / (kf->p + kf->r);              // 计算增益
    kf->x += kf->k * (measurement - kf->x);       // 校正状态
    kf->p *= (1 - kf->k);                         // 更新协方差
    
    return kf->x;
}
```

## 五、一阶滞后滤波

### 原理

通过加权系数α平衡新采样值与历史输出值。

### 应用场景

- 🔌 PWM电流高频噪声抑制
- 🌡️ 温度控制减少执行器频繁动作

### 代码实现



```
#define ALPHA 0.3  // 平滑系数(0<α<1)

float first_order_filter(float new_sample) {
    static float prev_output = 0;
    prev_output = ALPHA * new_sample + (1-ALPHA)*prev_output;
    return prev_output;
}
```

## 🧩 选型建议

|        应用场景        |       推荐算法        | 计算复杂度 |    适用系统    |
| :--------------------: | :-------------------: | :--------: | :------------: |
|  资源受限系统(8位MCU)  |  限幅滤波/中位值滤波  |   ★☆☆☆☆    | 低端嵌入式设备 |
|  动态系统控制(无人机)  |      卡尔曼滤波       |   ★★★★☆    |  高性能处理器  |
| 高频信号处理(电机电流) | 一阶滞后/滑动平均滤波 |   ★★☆☆☆    |  实时控制系统  |

> 注：所有代码示例均使用C语言实现，可直接移植到嵌入式平台



## 优化要点

1. 添加了图标符号增强可读性
2. 代码块使用独立语法标记
3. 表格增加了计算复杂度和适用系统列
4. 算法原理描述更精炼
5. 保留所有关键代码注释
6. 使用emoji符号标注应用场景类型
7. 添加全局注说明代码通用性