Pain Gesture 与 G1 共用动画控制器实现双人同步动作
一、背景与目标
在之前的工作中，已经为 Pain Gesture 机器人实现了语音控制行走、换装、七种舞蹈动作、跳跃、攀爬、慢跑等功能，所有动作均通过 Ctrl_Dance_Sprinkler 动画控制器进行管理。
场景中还保留着最初的 G1 机器人（前期已被禁用）。为了增强演示效果，我们希望在发出动作指令时，G1 能和 Pain Gesture 同时播放相同的动画，实现双人同步舞蹈、同步动作的效果。
核心思路是让两个机器人共用同一个动画控制器实例，语音指令触发时，同时向两个机器人的 Animator 组件发送相同的 Trigger。
二、准备工作
2.1 激活 G1 机器人
•	在 Unity 编辑器的 Hierarchy 窗口中找到 G1 物体。
•	选中 G1，在 Inspector 面板的最顶部，勾选其名称左侧的复选框，使其在场景中处于激活状态。
2.2 确认 G1 的组件配置
•	Animator 组件：G1 身上必须挂载 Animator 组件。如果之前被移除，需要重新添加（Add Component → Animator）。
•	Avatar：G1 的 Animator 组件中的 Avatar 字段应保持为 G1 自己的 Avatar，不要修改。
•	Controller 字段暂时可以为空，后续由脚本自动同步，无需手动拖拽。
•	G1 不需要 NavMeshAgent 和 Wardrobe 等组件（如果只需要它跟随动作，不参与寻路和换装）。
2.3 确认 Pain Gesture 的动画控制器
•	选中 Pain Gesture，查看其 Animator 组件中的 Controller 字段，确认为 Ctrl_Dance_Sprinkler（该控制器中已配置好所有动作的状态和过渡）。
三、同步动画控制器的实现
3.1 技术原理
•	Unity 的 Animator.runtimeAnimatorController 属性允许在运行时动态更换动画控制器。
•	将 G1 的 runtimeAnimatorController 设置为与 Pain Gesture 相同的控制器实例，两个 Animator 就会拥有完全相同的状态机、参数和过渡逻辑。
•	随后向两个 Animator 设置相同的 Trigger 参数，即可触发相同的动画播放。
3.2 代码实现
打开 SpeechToTextAndLLM.cs，进行以下修改。
（1）添加 G1 的 Animator 引用
在类的成员变量区域添加：
csharp
private Animator g1Animator;   // G1 的 Animator 组件
（2）在 Start() 中初始化并同步控制器
在 Start() 函数中，获取 "G1" 物体并同步其动画控制器：
csharp
// 初始化 G1 并同步动画控制器
GameObject g1Obj = GameObject.Find("G1");
if (g1Obj != null)
{
    g1Animator = g1Obj.GetComponent<Animator>();
    if (g1Animator != null && animator != null)
    {
        // 让 G1 共用 Pain Gesture 的动画控制器
        g1Animator.runtimeAnimatorController = animator.runtimeAnimatorController;
        Debug.Log("G1 动画控制器已同步");
    }
    else
    {
        Debug.LogWarning("G1 或 Pain Gesture 缺少 Animator 组件，无法同步");
    }
}
else
{
    Debug.LogWarning("场景中未找到 G1 机器人");
}
（3）创建辅助方法，同时触发两个机器人的动作
为了避免在每处动作触发代码中重复写两遍，添加一个私有方法：
csharp
/// <summary>
/// 同时向 Pain Gesture 和 G1 的 Animator 设置同一个 Trigger
/// </summary>
private void TriggerBoth(string triggerName)
{
    if (animator != null) animator.SetTrigger(triggerName);
    if (g1Animator != null) g1Animator.SetTrigger(triggerName);
}
（4）修改 ProcessActionCommand 方法
将原来所有 animator.SetTrigger(...) 的调用全部替换为 TriggerBoth(...)。
例如，桑巴舞指令从：
csharp
if (userInput.Contains("桑巴"))
{
    animator.SetTrigger("Samba");
    PlayTextToSpeech("来段桑巴");
    return true;
}
改为：
csharp
if (userInput.Contains("桑巴"))
{
    TriggerBoth("Samba");
    PlayTextToSpeech("来段桑巴");
    return true;
}
对于随机舞蹈指令，同样使用 TriggerBoth(pick) 替换原来的 animator.SetTrigger(pick)。
需要替换的动作包括：七种舞蹈（Samba, Silly, House, RobotHop, Rumba, Salsa, Twist）、跳跃（Jump）、攀爬（Climb）、慢跑（Jog）、向右转（RightTurn）、谢谢/鞠躬（Thankful）。
（5）其他功能保持不变
•	换装、行走、帽子等指令目前只控制 Pain Gesture，不涉及 G1，因此无需修改。
•	如果需要 G1 也参与换装，可后续单独添加对 G1 的 Wardrobe 控制，但非必需。
________________________________________
四、完整流程回顾与运行测试
4.1 操作清单
1.	在 Hierarchy 中激活 G1。
2.	确保 G1 上挂载了 Animator 组件（Avatar 保持原样）。
3.	确保 Pain Gesture 的 Animator 组件中 Controller 为 Ctrl_Dance_Sprinkler。
4.	将修改后的 SpeechToTextAndLLM.cs 脚本保存，等待 Unity 编译完成。
4.2 运行测试
•	点击 Unity 的 Play 按钮。
•	对着麦克风说出以下指令，观察两个机器人是否同步动作：
语音指令	预期效果
“桑巴”	Pain Gesture 和 G1 同时开始跳桑巴舞
“跳个舞”	两个机器人同时随机播放一支舞蹈
“向右转”	两个机器人同时向右转身
“谢谢”	两个机器人同时鞠躬感谢
“去客厅”	Pain Gesture 走向客厅，G1 留在原地（或停止舞蹈返回待机）
“穿上红色衣服”	只有 Pain Gesture 换装，G1 不受影响
4.3 常见问题及处理
问题现象	可能原因	解决方法
G1 不动，Pain Gesture 正常	G1 的 Animator 组件丢失或 g1Animator 为 null	检查 G1 是否挂载 Animator，控制台是否有 “G1 动画控制器已同步” 日志
G1 动作与 Pain Gesture 不同步	G1 的 Animator 中 Apply Root Motion 开启导致位移	取消 G1 的 Animator 组件中 Apply Root Motion 的勾选
两个机器人位置重叠	初始位置太近	在运行前将 G1 的 Transform Position 的 X 轴偏移几米
G1 跳舞时到处乱跑	动画自带的根运动（Root Motion）导致位移	在 G1 的 Animator 组件中取消 Apply Root Motion 勾选
五、总结
通过让 G1 在运行时动态共用 Pain Gesture 的动画控制器实例，并利用一个统一的 TriggerBoth 方法同时设置两个 Animator 的 Trigger，成功实现了两个机器人同步执行舞蹈、跳跃、鞠躬等动作的功能。该方法无需修改 Animator 控制器的结构，也不需要为 G1 单独配置复杂的状态机，代码改动量小，逻辑清晰，便于后续继续扩展更多角色。

