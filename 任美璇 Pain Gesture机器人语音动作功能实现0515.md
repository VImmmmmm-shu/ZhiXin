Pain Gesture机器人语音动作功能实现
一、背景与目标
在已实现语音控制行走、换装、帽子功能的基础上，进一步扩展交互能力，使机器人能够根据语音指令完成三个新动作：跳跃（Jump）、攀爬（Freehang Climb） 和绕圈慢跑（Jog In Circle）。所有动作均需通过 Mixamo 动画库获取，并在 Unity 中与现有的 Humanoid 骨架和语音控制系统无缝集成。
二、动画获取与导入
2.1 从 Mixamo 下载动画
•	打开 Mixamo，先选择一个 Humanoid 角色（例如 Sporty Granny 或任意标准人形角色），再依次选择三个动画：
o	Jump（跳跃）
o	Freehang Climb（悬挂攀爬）
o	Jog In Circle（绕圈慢跑）
•	下载设置：Format 选 FBX Binary(.fbx)，FPS 为 30，Skin 选 With Skin，Keyframe Reduction 设为 none。
•	将下载的三个 .fbx 文件拖入 Unity 的 Assets/Animations 文件夹。
2.2 动画 Rig 配置
•	在 Project 窗口依次选中每个动画文件，在 Inspector 的 Rig 标签页中设置：
o	Animation Type：Humanoid
o	Avatar Definition：根据情况选择
	若动画自带完整骨骼且与 Pain Gesture 兼容，可选择 Copy From Other Avatar，并将 Pain Gesture 拖入 Source。
	常见问题：Freehang Climb 在选择 Copy From Other Avatar 时出现错误 Invalid copied Avatar Rig Configuration: No human bone found。原因是该动画文件不含完整的骨骼定义。
	解决方案：将 Avatar Definition 改为 Create From This Model，Unity 会单独为此动画生成 Humanoid Avatar，不影响最终使用（只要角色本身是 Humanoid 就能正确重定向）。
o	点击 Apply。
•	在 Animation 标签页中，根据需求设置循环：
o	Jump：不勾选 Loop Time（单次跳跃）
o	Freehang Climb：不勾选 Loop Time（单次攀爬）
o	Jog In Circle：勾选 Loop Time（循环慢跑，直到被新指令打断）
三、在 Animator Controller 中配置动作
3.1 添加动画状态
•	双击 Pain Gesture 使用的动画控制器 Ctrl_Dance_Sprinkler，打开 Animator 窗口。
•	将三个动画文件直接拖入 Animator 窗口，生成三个新的灰色状态：
o	Jump
o	Freehang Climb
o	Jog in Circle
•	拖入后状态默认可能显示为 mixamo.com，可在右侧 Inspector 顶部手动修改状态名称，或在 Project 窗口中展开动画文件，将内部的 Animation Clip 重命名后刷新 Animator。
3.2 创建触发参数
•	在 Animator 窗口左上角切换到 Parameters 标签页。
•	点击 + → Trigger，分别创建三个 Trigger 参数：
o	Jump
o	Climb
o	Jog
3.3 设置动作过渡
•	从任意状态触发动作：
o	右键 Any State → Make Transition，拖到 Jump 状态，取消勾选 Has Exit Time，添加条件 Jump。
o	同上，创建 Any State → Freehang Climb（条件 Climb），Any State → Jog in Circle（条件 Jog）。
•	动作结束返回待机/移动：
o	右键 Jump → Make Transition，拖到 Movement（待机/行走混合树），勾选 Has Exit Time，不加条件。
o	同样设置 Freehang Climb → Movement、Jog in Circle → Movement。
•	最终结构：
text
Any State ─(Jump)──→ Jump ─(Exit Time)──→ Movement
Any State ─(Climb)─→ Freehang Climb ─(Exit Time)──→ Movement
Any State ─(Jog)───→ Jog in Circle ─(Exit Time)──→ Movement
四、语音控制脚本改造
4.1 添加动作处理函数
在 SpeechToTextAndLLM.cs 中新增以下代码：
•	类成员变量：private Animator animator;
•	在 Start() 中获取组件：animator = g1Robot.GetComponent<Animator>();
•	动作判断函数 ProcessActionCommand：
csharp
bool ProcessActionCommand(string userInput) {
    if (animator == null) return false;
    if (userInput.Contains("跳") || userInput.Contains("跳跃")) {
        animator.SetTrigger("Jump");
        PlayTextToSpeech("好的，跳！");
        return true;
    }
    if (userInput.Contains("攀爬") || userInput.Contains("爬")) {
        animator.SetTrigger("Climb");
        PlayTextToSpeech("开始攀爬");
        return true;
    }
    if (userInput.Contains("慢跑") || userInput.Contains("跑步") || userInput.Contains("绕圈")) {
        animator.SetTrigger("Jog");
        PlayTextToSpeech("开始慢跑");
        return true;
    }
    return false;
}
4.2 集成到指令分发流程
•	在 StopRecording 的 SendToLLM 回调中优先处理动作指令，避免被后续换装/移动逻辑误处理：
csharp
StartCoroutine(SendToLLM(processed, llmResp =>
{
    if (!ProcessActionCommand(processed))
    {
        // 原有的换装、移动等逻辑
        bool composite = processed.Contains("衣服") && processed.Contains("帽子");
        bool hatDone = false;
        if (!composite) hatDone = ProcessHatCommand(processed);
        if (!hatDone || composite) ProcessClothingCommand(llmResp, processed);
        ProcessMoveCommand(llmResp, processed);
    }
    resultText.text = "LLM: " + llmResp;
    PlayTextToSpeech(llmResp);
}, err => resultText.text = "LLM失败: " + err));
•	关键词扩展：在 ContainsValidKeywords 中添加动作相关词（"跳","攀爬","爬","跑步","慢跑","绕圈"），确保指令不会被提前过滤。
4.3 语音识别纠错调整
•	百度语音可能将“跳”识别为“挺跳跃”，在 CorrectAndInferIntent 中增加清洗：
csharp
raw = raw.Replace("挺跳跃", "跳跃");
•	保留之前的“简称在”“放下”等误识别清洗，防止干扰。

五、最终效果与总结
5.1 已完成功能
•	语音指令“跳” → 角色执行一次跳跃动画，落回待机状态。
•	语音指令“攀爬” → 角色悬挂攀爬，结束后自动回到待机。
•	语音指令“慢跑” → 角色循环绕圈慢跑，直到下一条指令打断。
•	原有换装、帽子、行走功能完全保留，各指令互不干扰。
5.2 技术要点
•	Humanoid 动画重定向允许不同骨架的动画通用，只要 Avatar 类型一致。
•	Any State 过渡使指令响应无需等待当前动画结束。
•	事务性添加功能时，应逐步修改并验证，避免一次性改动导致全局失效。
•	语音指令分发应遵循优先级顺序，动作指令优先于换装和移动，避免关键词重叠导致误判。
5.3 后续扩展方向
•	可继续添加更多 Mixamo 动作（如跳舞、挥手、坐下）。
•	优化语音纠错库，覆盖更多同音词和方言变体。
•	结合动作播放时的导航中断逻辑，使角色动作与移动更加协调。




通过调整摄像头的视野，有些时候效果会更好。
我现在调整到68.7比之前的40要好很多
 
