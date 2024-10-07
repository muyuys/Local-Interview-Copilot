# Interview Copilot

使用本地funsar语音识别+本地qwen2语言模型识别

## Developing
This project is based on Vue2. Just  `cd app`:

**install packages:** `npm install`

**develop:** `npm run serve`

**build:** `npm run build`


## 本地的语音识别
来自 https://github.com/0x5446/api4sensevoice
代码里配置了使用的语言为中文


## 本地模型运行
使用ollama，示例使用qwen2:7b 模型
在setting中配置apiKey=ollama
使用默认端口 11434

## 远程模型
openai支持的都可以，增加了智谱的glam-4-flash，国内免费试用

# 更新日志
- 2024.10.07
    - 增加了funasr模型的支持
    - 引入了https://github.com/xiangyuecn/Recorder
    - 修改语音识别区域文本为可编辑

