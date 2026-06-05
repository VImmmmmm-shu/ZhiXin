为 Pain Gesture 机器人配置舞蹈音乐并实现静默跳舞
一、任务背景
在已实现语音控制多机器人同步舞蹈（七种舞种：桑巴、搞笑舞、浩室、机器人街舞、伦巴、萨尔萨、扭扭舞）的基础上，需要为每一种舞蹈配置对应的背景音乐，并在跳舞时暂停大模型的语音回复，避免人声干扰音乐播放，提升现场演示的沉浸感。
二、获取免费音乐资源
从以下免费音效网站下载与各舞蹈风格匹配的纯音乐（无歌声），格式为 MP3 或 WAV：
平台	网址	特点
5sing 原创音乐	5sing.kugou.com	国内原创 / 翻唱基地，独立音乐人自主上架歌曲，作者开放权限即可免费下载 MP3，古风、翻唱、伴奏资源超多，完全合规。
下载后将文件重命名，便于管理：
•	Samba_Music
•	Silly_Music
•	House_Music
•	RobotHop_Music
•	Rumba_Music
•	Salsa_Music
•	Twist_Music
三、在 Unity 项目中导入音乐文件
3.1 创建 Music 文件夹
1.	在 Unity 编辑器的 Project 窗口中找到 Assets 文件夹。
2.	右键点击 Assets，选择 Create → Folder，命名为 Music。
3.2 导入音乐文件
1.	在操作系统的文件管理器中，选中所有下载好的音乐文件。
2.	将它们直接拖入 Unity 的 Assets/Music 文件夹中。
3.	等待 Unity 自动导入，完成后可在 Project 窗口中看到音乐文件的图标。
3.3 验证音乐文件
•	在 Project 窗口中单击选中任意一个音乐文件。
•	查看 Inspector 面板底部的音频预览窗口，点击 ▶ 播放按钮，确认能正常听到声音。
•	如果听不到声音或出现噪音，说明文件编码可能不兼容，需用 Audacity 等音频软件重新导出为 WAV 格式（PCM 16-bit, 44100 Hz）。
四、修改语音控制脚本，关联音乐与舞蹈
打开 SpeechToTextAndLLM.cs，进行以下修改。
4.1 添加音乐相关成员变量
在类的顶部声明区域加入：
csharp
[Header("舞蹈音乐")]
public AudioClip[] danceMusics;   // 索引: 0=Samba, 1=Silly, 2=House, 3=RobotHop, 4=Rumba, 5=Salsa, 6=Twist
private AudioSource musicSource;  // 专门播放舞蹈音乐的 AudioSource
4.2 在 Start() 中初始化音乐播放器
csharp
// 初始化舞蹈音乐播放器
musicSource = gameObject.AddComponent<AudioSource>();
musicSource.loop = true;        // 音乐循环播放
musicSource.playOnAwake = false;
musicSource.volume = 0.6f;      // 音量（0.0 ~ 1.0），可根据需要调整
musicSource.priority = 0;       // 最高优先级，保证音乐流畅
musicSource.outputAudioMixerGroup = null;  // 使用默认输出，避免被静音
4.3 添加播放音乐的方法
csharp
private void PlayDanceMusic(int index)
{
    if (danceMusics == null || index < 0 || index >= danceMusics.Length)
        return;
    if (danceMusics[index] == null)
        return;

    if (musicSource.isPlaying)
        musicSource.Stop();

    musicSource.clip = danceMusics[index];
    musicSource.Play();
}
4.4 在舞蹈指令中调用音乐播放
在 ProcessActionCommand 方法中，为每一种舞蹈触发时添加 PlayDanceMusic(索引) 调用，并移除原有的 PlayTextToSpeech 语音播报，使舞蹈时机器人静默。
以桑巴舞为例：
csharp
// 桑巴舞
if (userInput.Contains("桑巴"))
{
    TriggerBoth("Samba");
    PlayDanceMusic(0);   // 播放桑巴音乐
    return true;
}
随机舞蹈指令同样需要获取随机索引并播放对应音乐：
csharp
if (userInput.Contains("跳个舞") || userInput.Contains("跳舞") || userInput.Contains("来段舞") || userInput.Contains("舞蹈"))
{
    string[] dances = { "Samba", "Silly", "House", "RobotHop", "Rumba", "Salsa", "Twist" };
    int index = UnityEngine.Random.Range(0, dances.Length);
    string pick = dances[index];
    TriggerBoth(pick);
    PlayDanceMusic(index);
    return true;
}
4.5 在 StopRecording 中屏蔽舞蹈时的 TTS 语音
修改大模型回调中的逻辑，只有非舞蹈指令才调用 PlayTextToSpeech：
csharp
StartCoroutine(SendToLLM(processed, llmResp =>
{
    bool handled = ProcessActionCommand(processed);
    if (!handled)
    {
        // 处理换装、帽子、移动
        bool composite = processed.Contains("衣服") && processed.Contains("帽子");
        bool hatDone = false;
        if (!composite) hatDone = ProcessHatCommand(processed);
        if (!hatDone || composite) ProcessClothingCommand(llmResp, processed);
        ProcessMoveCommand(llmResp, processed);
    }

    resultText.text = "LLM: " + llmResp;

    // 只有非舞蹈动作，才用TTS播报大模型回复
    if (!handled)
    {
        PlayTextToSpeech(llmResp);
    }
}, err => resultText.text = "LLM失败: " + err));
4.6 在开始录音时停止音乐
为了让音乐在用户点击“开始录音”时自动暂停，修改 StartRecording 方法：
csharp
if (musicSource != null && musicSource.isPlaying)
    musicSource.Stop();
这样每次录音时音乐都会停止，直到新的舞蹈指令触发再次播放。
五、在 Inspector 中配置音乐数组
1.	在 Hierarchy 中选中挂载 SpeechToTextAndLLM 脚本的物体（通常是 Canvas）。
2.	在 Inspector 中找到 Speech To Text And LLM (Script) 组件下的 Dance Musics 数组。
3.	将数组大小 Size 设置为 7。
4.	依次从 Assets/Music 文件夹中拖入对应的音乐文件：
o	Element 0 → 桑巴音乐（Samba_Music）
o	Element 1 → 搞笑舞音乐（Silly_Music）
o	Element 2 → 浩室音乐（House_Music）
o	Element 3 → 机器人街舞音乐（RobotHop_Music）
o	Element 4 → 伦巴音乐（Rumba_Music）
o	Element 5 → 萨尔萨音乐（Salsa_Music）
o	Element 6 → 扭扭舞音乐（Twist_Music）
六、调试与常见问题
•	排查 PlayDanceMusic 是否被调用：在方法开头添加 Debug.Log 日志，确认索引正确传入。
•	检查 Dance Musics 数组：确认对应索引的槽位不为空（不是 None）。
•	检查音频文件本身：在 Inspector 中点击预览播放按钮，确认能正常出声。
•	检查 musicSource 是否被意外静音：确保 musicSource.mute = false，volume > 0，enabled = true。

七、最终效果
•	用户说出“桑巴”“伦巴”等具体舞种指令，Pain Gesture、G1、G2 三个机器人同步舞蹈，并播放对应风格纯音乐。
•	用户说出“跳舞”或“跳个舞”，系统随机选择一种舞蹈并播放相应音乐。
•	跳舞时机器人不再发出 TTS 语音，只有音乐循环播放。
•	用户点击“开始录音”按钮时，音乐自动停止，方便发出下一条指令。
•	原有的换装、行走、跳跃等功能不受影响，仍保留语音反馈。
通过以上步骤，成功为舞蹈机器人配置了背景音乐，并优化了人机交互体验，使演示效果更加生动自然。

