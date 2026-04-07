# 在手机上用 Claude Code 写代码，我是怎么把飞书接上去的

故事是这样的。

我最近有一个很蠢但又很真实的焦虑，用 Claude Code 的朋友可能都懂，就是那种 token 用不完的焦虑。

对，你没看错。不是没 token 用焦虑，是用不完也焦虑。

Max 订阅一个月好几百刀，每天有一大堆 token 额度，但你总不能 24 小时坐在电脑前吧？通勤的时候、等人的时候、躺床上刷手机的时候，那些 token 就在那里安静地过期。你就眼睁睁看着它们消失。

然后我就想，能不能在手机上也用 Claude Code？

不是那种阉割版的 AI 对话，是完整的 Claude Code，能读文件、能跑命令、能改代码的那种。手机上发一句「帮我把那个 bug 修了」，它就真的去改代码了。

听起来有点离谱对吧。但我还真搞出来了。

## 怎么想到的

刷 GitHub 的时候看到一个叫 Claude-to-IM-skill 的项目，op7418 做的，1800 多个 star。这哥们做了一套 skill，能把 Claude Code 桥接到 Telegram、Discord、飞书、QQ、微信，基本上主流 IM 全覆盖了。

项目地址在这：https://github.com/op7418/Claude-to-IM-skill

我当时看到就觉得，卧槽这不就是我想要的东西吗。

然后就开始折腾。

## 实际操作

整个架构其实很简单，一句话就能说清楚：

```
你的手机飞书 → 飞书云端 ←WebSocket→ 你电脑上跑的桥接服务 → Claude Code
```

飞书 SDK 支持 WebSocket 长连接，不需要公网 IP，不需要域名，不需要 ngrok。你电脑开着就行。

我参考了 Claude-to-IM-skill 的思路，用飞书官方 SDK 加上 Claude Agent SDK，写了一个轻量版的桥接服务。核心代码就一个文件，bridge.mjs，200 来行。

关键的地方就两个 SDK：

`@larksuiteoapi/node-sdk` 负责跟飞书通信，WebSocket 长连接收消息、发回复。

`@anthropic-ai/claude-agent-sdk` 负责启动 Claude Code 会话。注意这个不是普通的 API 调用，它会起一个完整的 Claude Code 进程，能读写文件、跑 bash、用所有工具。跟你在终端里用 Claude Code 是一样的体验。

飞书那边需要做的事：

1. 去飞书开放平台创建一个自建应用，加上机器人能力
2. 加权限：im:message、im:message.group_at_msg、im:message.p2p_msg
3. 事件订阅加 im.message.receive_v1，投递方式选长连接
4. 发版审批

然后本地：

```bash
npm install @larksuiteoapi/node-sdk @anthropic-ai/claude-agent-sdk
```

启动：

```bash
FEISHU_APP_ID=你的appid \
FEISHU_APP_SECRET=你的secret \
AUTO_APPROVE=true \
node bridge.mjs
```

看到 `✅ WebSocket connected, listening...` 就算接上了。

## 踩的坑

坦率的讲，这个过程不算顺利。

第一个坑，Claude-to-IM-skill 直接 clone 下来 build 不过。它依赖一个叫 claude-to-im 的核心库，但这个库没发布到 npm，package.json 里写的是 file:../Claude-to-IM 指向本地路径，缺了 src/lib/bridge/ 整个目录。用 npx skills add 装也一样。所以我最后选择自己写了一个轻量版，核心逻辑是一样的，就是精简了很多。

第二个坑，飞书 SDK 的 API 变了。网上的教程和一些 skill 里写的是在 WSClient 构造函数里传 eventDispatcher，但新版 SDK 改成了在 start() 方法里传。就这么一个参数位置的变化，让我 debug 了一会儿。

第三个坑，飞书应用的权限审批。你加了权限之后必须发一个新版本让管理员审批，权限才生效。这个如果是自己的企业还好，秒批。

## 效果

现在我通勤的时候，掏出手机打开飞书，给机器人发一句话，它就在我电脑上跑 Claude Code 帮我干活。

私聊直接对话，群里 @机器人也行。支持上下文，多轮对话没问题。长消息自动分片。

最爽的是，AUTO_APPROVE=true 开了之后，它是全自动的。你说「帮我把那个函数重构一下」，它就真的去改文件了，不需要你确认权限。

当然这也意味着你得信任它，毕竟它拿到的是完整的文件系统访问权限。我自己用是无所谓，反正改坏了 git reset 就行。

## 相关链接

- Claude-to-IM-skill（推荐看看，完整版支持5个平台）：https://github.com/op7418/Claude-to-IM-skill
- 卡兹克老师的写作 skill（本文风格参考）：https://github.com/KKKKhazix/khazix-skills
- 我做的活人感写作 skill（基于卡兹克风格逆向工程）：https://github.com/siuserxiaowei/alive-writer-skill

以上，既然看到这里了，如果觉得不错，随手点个赞、在看、转发三连吧，
如果想第一时间收到推送，也可以给我个星标⭐～
谢谢你看我的文章，我们，下次再见。
