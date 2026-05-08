Pain Gesture 机器人换装与纹理功能实现流程
一、目标
在已实现机器人行走功能的基础上，完成语音控制换装功能。为完成“穿上真实的衣服”，而非简单的皮肤颜色替换。最终不仅要实现颜色切换，还要为衣服添加纹理，使其看起来更具布料质感。
二、模型导入与基础配置
2.1 模型导入
•	从 Mixamo 或其他资源平台下载穿着衣服的角色模型，格式为 FBX for Unity。
•	将 FBX 文件拖入 Unity 的 Project 窗口（Assets 文件夹）。
•	选中导入的模型，在 Inspector 的 Rig 标签页中将 Animation Type 设置为 Humanoid，点击 Apply。
2.2 场景配置
•	将模型拖入 Hierarchy，命名为 Pain Gesture。
•	调整 Transform 参数，特别要注意 Y 轴坐标，确保角色脚底刚好站在导航网格上，避免因悬浮或陷入地面而导致寻路失效。
2.3 添加寻路组件
•	选中 Pain Gesture，添加两个组件：
o	Nav Mesh Agent：设置 Speed=3.5，Stopping Distance=0，Radius=0.5，Height=2。
o	Nav Agent Animator：之前编写的脚本，用于将 NavMeshAgent 的速度传递给 Animator，驱动行走动画。
2.4 指定动画控制器
•	在 Animator 组件中，将 Controller 字段设置为项目中已有的动画控制器（如 Ctrl_Dance_FreeStyle），让角色能播放待机和行走动画。
三、语音控制系统适配
3.1 修改语音脚本指向
•	打开 SpeechToTextAndLLM.cs，找到 Start() 函数中的：
csharp
g1Robot = GameObject.Find("G1");
•	将其修改为：
csharp
g1Robot = GameObject.Find("Pain Gesture");
3.2 配置摄像机跟随
•	找到 Main Camera 上挂载的跟随脚本，将每个脚本的 Target 字段由 G1 替换为 Pain Gesture。
3.3 行走测试
•	运行游戏，通过语音指令“去客厅”测试角色是否能沿导航网格正确行走。
•	如果出现碰墙或不走的情况，排查原因：可能是 Y 轴坐标导致脚陷入地下，或者 NavMeshAgent 参数需要微调。
四、换装功能实现（材质替换方案）
4.1 初始方案：材质颜色替换
•	给 Pain Gesture 挂载原有的 Wardrobe 脚本。
•	在 Body Materials 数组中依次拖入四个不同颜色的材质球：RedCloth、BlueCloth、GrayCloth、GreenCloth。
•	通过语音指令“穿上红色衣服”等来测试颜色切换。
4.2 分区换装实现（只换衣服，不动眼镜和五官）
•	问题描述：在完成基础换装后，发现角色身着红色衣服时，眼镜、眉毛、嘴巴等也变得通体通红，缺乏真实感。
•	解决方案：修改 Wardrobe 脚本，使其仅替换身体网格上的衣服材质，而保留眼镜、五官的原始材质。
•	实现步骤：
1.	定位身体渲染器：确认 Pain Gesture 的子物体中，Fitness_Grandma_BodyGeo 控制身体，且它身上挂载了 Skinned Mesh Renderer 组件。该组件拥有两个材质槽：Element 0（衣服）和 Element 1（眼镜）。
2.	修改 Wardrobe 脚本逻辑：将原本“遍历所有渲染器并替换其所有材质”的逻辑，改为“只在 bodyRenderer 上替换其第一个材质槽（Element 0）”，从而确保眼镜材质不被修改。
3.	配置 Inspector：在 Pain Gesture 的 Wardrobe 组件中，将 Fitness_Grandma_BodyGeo 拖入 Body Renderer 字段。眼镜材质（Lens_MAT）复制一份并调整 Albedo 为黑色，拖入 Element 1 中，实现黑色眼镜框。
•	最终效果：成功实现分区换装，眼镜保持黑色，眉毛、眼睛、嘴巴保留原始模样。
五、为衣服添加纹理
5.1 关键发现与尝试
•	发现：观察原图，发现帽子上有布料纹理，推断帽子和衣服共用同一张身体材质贴图。
•	尝试恢复失败：检查身体材质 Grandma_MAT，发现其 Albedo 贴图槽为灰色（无贴图），无法直接利用原生纹理，需要自己创建。
5.2 自己制作纹理贴图
•	遇到的问题：在 Project 窗口右键菜单里找不到 Texture 2D 选项。
•	解决方案：改用 Windows 自带“画图”工具手动制作一张 PNG 图片。
•	导入 Unity 并应用：
1.	将 ClothTexture.png 拖入 Unity 的 Material 文件夹。
2.	选中红色衣服材质球 RedCloth，在 Inspector 的 Albedo 左侧小方框中拖入 ClothTexture。
3.	点击 Albedo 右侧颜色块，调成红色倾向，让纹理显现。
•	验证：场景中 Pain Gesture 的衣服变成了带条纹纹理的红色，不再是光滑纯色。切换其他颜色时，对其他材质球重复相同操作即可。

六、遇到的语音识别错误及解决方案
•	误识别“穿上”为“简称在”：在 CorrectAndInferIntent 函数最前面添加脏词清洗，将“简称在”等词过滤或映射为正确指令。
•	识别速度慢：将 StartRecording 中的录音参数由 10 秒、44100 Hz 改为 5 秒、16000 Hz，数据量缩小至原来的 1/12，识别速度明显提升。
•	偶发 3307 错误：在识别失败时增加自动重试机制，等待 0.3 秒后重新上传音频。
七、最终成果与总结
7.1 已完成的功能
•	语音控制行走（去客厅、卧室、厨房），沿导航网格正常移动。
•	摄像机跟随新角色。
•	语音控制换装（红/蓝/灰/绿四色切换），且眼镜保持黑色，五官不受影响。
•	衣服自带条纹纹理，不再是光滑纯色。
•	模型切换技术框架已搭建完毕，可随时替换带贴图模型实现升级。
7.2 技术要点
•	纯色换装通过材质替换实现，分区控制则通过限定替换特定材质槽来完成。
•	当模型无贴图时，可通过自制纹理贴图并应用到材质球上来增强视觉效果。
•	语音识别的准确性和速度是保证系统稳定性的关键，通过优化录音参数和添加纠错逻辑可有效改善。
•	模型切换方案证明了技术可行性，其瓶颈在于素材本身而非代码逻辑。
