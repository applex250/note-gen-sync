先在 A 机上核实分割/检测所用去雾权重的来源（fawnetLA_* 目录是谁生成的、用什么训练的），确保回复信里不写错：

关键发现：分割用的 FAWNet 权重是 `fawnet_lovela_heavy` —— 在雾化 LoveDA 上训练的。再确认三基线是否同样训练、以及检测用的什么权重：

全部从服务器核实完毕（训练集目录、ckpt 命名、雾配置都查了原始文件）。**重雾档**三任务如下：

---

### ① 语义分割


| 项      | 内容                                                                                             |
| ------ | ---------------------------------------------------------------------------------------------- |
| 去雾器训练集 | **雾化 LoveDA 训练集**（2322 train + 200 val 的 hazy/GT 对，重雾档），4 法（FAWNet/SFRDP/MABDT/DR3DF）各自同协议独立训练 |
| 去雾器测试集 | 雾化 LoveDA 验证集 **500 张**                                                                        |
| 下游模型   | **D2LS**（ConvNeXt-B 骨干，LoveDA 预训练），冻结不微调                                                       |
| 下游数据集  | **LoveDA**（7 类地物）                                                                              |
| 加的雾    | **大气散射模型合成雾**（mkfog 引擎）：I=J·t+A(1−t)，透射率由伪深度调制，重雾档 β∈[0.9, 1.2] 逐图随机，训练/测试同配置                  |
| 重雾结果   | Clean .6323 / 雾 .2737 / **FAWNet .5968** / MABDT .5825 / SFRDP .5703 / DR3DF .5399             |


### ② 旋转框目标检测


| 项      | 内容                                                                                           |
| ------ | -------------------------------------------------------------------------------------------- |
| 去雾器训练集 | **DOTA-v1.0 train** 切成 512×512（6000 张 clean）→ 雾化后 5700 train + 300 val（重雾档），4 法同协议独立训练       |
| 去雾器测试集 | 雾化 **DOTA-v1.0 val** 500 张切片（判官从未见过 val）                                                     |
| 下游模型   | **YOLOv8x-obb**（DOTA-v1.0 预训练），冻结不微调                                                         |
| 下游数据集  | **DOTA-v1.0**（15 类旋转框）                                                                       |
| 加的雾    | 同①大气散射雾，重雾 β∈[0.9, 1.2]                                                                      |
| 重雾结果   | Clean .6797 / 雾 .5836 / **FAWNet .6768** / SFRDP .6676 / DR3DF .6575 / MABDT .6510（mAP50-95） |


### ③ 高程读取


| 项      | 内容                                                                                                                                   |
| ------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| 去雾器训练集 | **雾化 RS3DBench 山地子集训练集**（2160 张 clean → veil 重雾 β1.5 → 2052 train + 108 val），4 法同协议各训 40 epoch                                       |
| 去雾器测试集 | 雾化山地测试集 **240 张**（veil β1.5）                                                                                                         |
| 下游模型   | **MoG2-Aerial**（MoGe2 ViT-L + 航空 LoRA，ECCV'26 AerialMetric），冻结不微调                                                                    |
| 下游数据集  | **RS3DBench**（瑞士 0.5m 正射影像）山地子集，16-bit DEM 高程真值                                                                                      |
| 加的雾    | **veil 云幕雾**（俯视图专用设计）：I=J·t+A(1−t)，t=max(e^(−β·s), 0.12)，s=fBm 云厚场（逐图定种子），A=0.92，重雾 **β=1.5**；**不用深度调制**（俯视图上伪深度与几何无关，深度调制雾会被判官“读穿”） |
| 评测协议   | 固定符号 Spearman ρ（符号约定 clean 集一次定死）                                                                                                    |
| 重雾结果   | Clean .5030 / 雾 .2905（27.9% 方向反转）/ **FAWNet .4678**（反转率回落 18.3%≈判官底噪 17.1%）/ SFRDP .4581 / MABDT .4568 / DR3DF .4370                 |


---

**统一要点**（回复信可直接写）：三任务协议一致——每任务将**该数据集训练划分**雾化后，4 个去雾法**同配方独立重训**（无测试泄漏），去雾结果喂给**冻结的**该领域 SOTA 判官（D2LS / YOLOv8x-obb / MoG2-Aerial），与雾输入、clean 输入三方对照；FAWNet 在三任务重雾档均第一，恢复率 90.1% / 97.0% / 83.4%。  
审稿人意见：

3. Although the experiments cover different haze densities, real remote sensing images may simultaneously suffer from haze, sensor noise, blur, and illumination variations. Additional experiments involving mixed degradation conditions would help establish the robustness and practical applicability of the proposed method.

6. The manuscript emphasizes the importance of preserving roads, buildings, and small objects in remote sensing imagery, but the evaluation focuses primarily on image restoration metrics. Additional object detection or semantic segmentation experiments using restored images would help demonstrate whether the improvements translate into practical benefits for remote sensing applications.  
  
  
F:\txp001\retudata002\r1-6 是三类下游任务的效果，我们将审稿人3点和6点 联合起来阐述，（上面加了什么只写了一部分，还缺少噪音和模糊描述 都是加了的），我们为了回应审稿人3的意见 给各个下游任务除了加上常规的合成雾 还加上了  
Sequential composite degradation: 7×7 isotropic Gaussian blur followed by zero-mean additive white Gaussian noise with $\sigma_n=7$.  
Light-to-moderate composite degradation consisting of slight blur plus low-intensity noise.  
When applied to hazy images, its relative effect is amplified due to the low contrast of hazy images, and it can be regarded as a light-to-moderate robust disturbance.  
模糊是在生成雾气的非均匀部分，  
  
  
  


[README.MD](http://README.MD)

## **对比位置（这点很关键）**


|             | **参数**     | **相对 S3** |
| ---------------- | ---------- | --------- |
| **FAWNet-S3轻量版** | **0.85 M** | 1         |
| EMPF-Net         | 0.30 M     | 0.35×     |
| DehazeFormer-s   | 1.28 M     | 1.5×      |
| **DEA-Net**      | **2.10 M** | **2.5×**  |
| FAWNet 原模型       | 9.28 M     | 10.9×     |


算力是 24.5 GMAC（教师 264.1 的 **1/10.8**）。所以对外表述应该是"**0.85 M 参数 / 24.5 GMAC，比 DEA-Net 小 2.5 倍、快 3.2 倍**"  
我们在实际的  


## Main Results (PSNR-Y (dB) / SSIM-Y)


| Dataset   | SateHaze1K-thin                                                                 | SateHaze1K-moderate                                                             | SateHaze1K-thick                                                                | RSID                                                                            | RICE1              | RICE2          | CUHK-CR2                                                                        |
| --------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- | ------------------ | -------------- | ------------------------------------------------------------------------------- |
| FAWNet    | 28.22 / 0.9789                                                                  | 31.53 / 0.9812                                                                  | 25.31 / 0.9498                                                                  | 28.71 / 0.9759                                                                  | **39.48** / 0.9918 | 38.59 / 0.9291 | 26.35 / 0.7549                                                                  |
| FAWNet-S3 | <span data-notegen-color="#f9fafb" style="color: #f9fafb">26.75 / 0.9716</span> | <span data-notegen-color="#f9fafb" style="color: #f9fafb">30.03 / 0.9774</span> | <span data-notegen-color="#f9fafb" style="color: #f9fafb">24.16 / 0.9410</span> | <span data-notegen-color="#f9fafb" style="color: #f9fafb">27.34 / 0.9694</span> | 38.73 / 0.9913     | 38.12 / 0.9191 | <span data-notegen-color="#f9fafb" style="color: #f9fafb">25.81 / 0.7099</span> |


&nbsp;

我们为实现轻量级边缘设备的部署，针对我们原模型做了极致的轻量级优化，在7个数据集下 能达到仅次于fawnet的效果 在整个图中排第二梯队  
  
同时我们对我们的fawnet模型做了多项下游实验都达到了不错的效果  
  
本论文的模型代码和轻量化部署代码会在论文录取后上传开源  
  
