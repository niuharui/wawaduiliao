📺 项目简介
点对点、根据距离、男女匹配的不记名聊天网页的实现
服务器不保留任何信息
所有聊天仅存在本地



🚨 重要声明
本项目仅供学习和个人使用
请勿将部署的实例用于商业用途或公开服务
如因公开分享导致的任何法律问题，用户需自行承担责任
项目开发者不对用户的使用行为承担任何法律责任

#  重要提示： 这是一个基础的部署指南，适用于测试和演示。在生产环境中，您需要考虑更多因素，如安全性（身份验证、授权、防止DDoS攻击）、性能优化、日志记录、监控、数据持久化、负载均衡等。

# 第一步：在阿里云购买和配置ECS服务器
登录阿里云控制台: 访问 https://home.console.aliyun.com/ 并使用您的账号登录。
购买ECS实例:
在控制台首页或服务列表中找到“云服务器ECS”。
点击“创建实例”。
选择配置（以下为示例配置，可根据需求调整）：
地域和可用区: 选择靠近您或您用户群的地域。
实例: 选择入门级（如ecs.t6-c1m1.large）或通用型（如ecs.g7a-c1m1.large）实例，1核2GB内存通常足够测试。
镜像: 选择“公共镜像” -> “Ubuntu” (例如 Ubuntu 22.04 LTS) 或 “CentOS” (例如 CentOS Stream 8)。本示例使用 Ubuntu。
存储: 系统盘选择SSD云盘，容量40GB通常足够。
网络和安全组:
选择或创建一个安全组。
确保安全组规则开放了必要的端口：
22/tcp (SSH访问)
80/tcp (HTTP, 可选)
443/tcp (HTTPS, 可选，用于wss)
8080/tcp (我们将用于WebSocket服务)
ICMP (用于ping测试，可选)
网络计费模式通常选择“按流量计费”。
公网IP: 选择“分配公网IPv4地址”。
实例名称: 给实例起个名字，例如 chat-app-server。
登录凭证: 设置实例登录密码或使用密钥对（推荐更安全）。
点击“立即购买”并完成支付流程。
获取公网IP地址:
实例创建完成后，在ECS控制台的实例列表中找到您的实例。
记下分配给它的“公网IP地址”。
# 第二步：连接到ECS服务器并安装环境
使用SSH连接: 使用终端（Mac/Linux）或PuTTY（Windows）通过SSH连接到您的ECS实例。
ssh root@<YOUR_ECS_PUBLIC_IP>
#或者如果您使用密钥对
#ssh -i /path/to/your/key.pem root@<YOUR_ECS_PUBLIC_IP>
(将 <YOUR_ECS_PUBLIC_IP> 替换为您的实际公网IP)
更新系统包 (Ubuntu):
apt update && apt upgrade -y
安装Node.js和npm:
#安装 NodeSource 仓库以获取较新版本的 Node.js
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
apt-get install -y nodejs
#验证安装
node -v
npm -v
# 第三步：部署WebSocket服务器
创建工作目录:
mkdir /home/chat-app
cd /home/chat-app
初始化Node.js项目并安装依赖:
npm init -y
npm install ws
创建WebSocket服务器代码 (server.js):
const WebSocket = require('ws');

// --- 配置 ---
const PORT = 8080; // WebSocket服务器监听的端口

// --- 状态 ---
const connectedClients = new Map(); // 存储连接的客户端 { ws: WebSocket, user: userInfo }

// --- 创建WebSocket服务器 ---
const wss = new WebSocket.Server({ port: PORT });

console.log(`WebSocket服务器启动，监听端口 ${PORT}`);

// --- 处理新连接 ---
wss.on('connection', (ws) => {
    console.log('新客户端连接');

    // 发送欢迎信息和当前用户列表
    ws.send(JSON.stringify({ type: 'welcome', message: 'Connected to server' }));

    // 当服务器收到客户端发来的消息
    ws.on('message', (message) => {
        try {
            const data = JSON.parse(message);
            console.log('收到消息:', data);

            switch (data.type) {
                case 'join':
                    // 用户加入
                    const userInfo = data.user;
                    if (userInfo && userInfo.id) {
                        connectedClients.set(ws, userInfo);
                        console.log(`用户 ${userInfo.name} (${userInfo.id}) 加入`);

                        // 通知所有其他客户端有新用户加入
                        const joinNotification = {
                            type: 'userJoined',
                            user: userInfo
                        };
                        broadcast(joinNotification, ws); // 不广播给发送者自己

                        // 向新用户发送当前在线用户列表
                        const userList = Array.from(connectedClients.values());
                        ws.send(JSON.stringify({
                            type: 'userList',
                            users: userList
                        }));
                    }
                    break;
                case 'message':
                    // 转发聊天消息给所有其他客户端
                    // 注意：在真实P2P中，这里会查找目标用户并直接发送
                    // 此处简化为广播，由客户端根据性别和距离过滤
                    const senderInfo = connectedClients.get(ws);
                    if (senderInfo) {
                        const messageToBroadcast = {
                            type: 'message',
                            fromId: senderInfo.id,
                            content: data.content,
                            timestamp: data.timestamp
                        };
                        broadcast(messageToBroadcast, ws); // 不广播给发送者自己
                    }
                    break;
                default:
                    console.log('未知消息类型:', data.type);
            }
        } catch (e) {
            console.error('处理消息时出错:', e, message);
        }
    });

    // 当客户端断开连接
    ws.on('close', () => {
        const userInfo = connectedClients.get(ws);
        if (userInfo) {
            connectedClients.delete(ws);
            console.log(`用户 ${userInfo.name} (${userInfo.id}) 离开`);
            // 通知所有其他客户端有用户离开
            const leaveNotification = {
                type: 'userLeft',
                userId: userInfo.id
            };
            broadcast(leaveNotification);
        }
    });

    // 处理连接错误
    ws.on('error', (error) => {
        console.error('WebSocket连接错误:', error);
        const userInfo = connectedClients.get(ws);
        if (userInfo) {
            console.log(`与用户 ${userInfo.name} (${userInfo.id}) 的连接出错`);
        }
    });
});

// --- 工具函数 ---

// 广播消息给所有连接的客户端（除了指定的排除者）
function broadcast(message, excludeWs = null) {
    const messageString = JSON.stringify(message);
    connectedClients.forEach((userInfo, clientWs) => {
        if (clientWs !== excludeWs && clientWs.readyState === WebSocket.OPEN) {
            clientWs.send(messageString);
        }
    });
}

// --- 优雅关闭 ---
process.on('SIGINT', () => {
    console.log('收到 SIGINT 信号，正在关闭服务器...');
    wss.close(() => {
        console.log('WebSocket服务器已关闭');
        process.exit(0);
    });
});
进入代码分屏
运行服务器:
node /home/chat-app/server.js
您应该会看到 WebSocket服务器启动，监听端口 8080 的输出。保持此终端窗口打开。
# 第四步：修改并部署前端代码
修改前端HTML代码中的WebSocket地址:
将您之前生成的HTML代码保存为本地文件，例如 index.html。
找到 const WEBSOCKET_SERVER_URL = 'wss://your-websocket-server.com'; 这一行。
将其修改为指向您的阿里云ECS服务器的公网IP和端口：
const WEBSOCKET_SERVER_URL = 'ws://<YOUR_ECS_PUBLIC_IP>:8080';
(将 <YOUR_ECS_PUBLIC_IP> 替换为您的实际公网IP)
注意: 这里使用的是 ws:// (非加密)。如果您的页面通过HTTPS (https://) 加载，浏览器会阻止连接到 ws://。为了解决这个问题，您需要配置SSL证书并使用 wss://。这通常涉及使用Nginx等反向代理。为简化起见，我们先使用 ws://，这意味着您需要通过HTTP (http://) 访问您的HTML页面。
上传前端文件到ECS (可选，也可以本地测试):
如果您想直接从ECS服务器访问前端页面，可以将 index.html 上传到ECS。
方法一：使用SCP命令 (本地终端):
scp /path/to/your/local/index.html root@<YOUR_ECS_PUBLIC_IP>:/home/chat-app/
方法二：在ECS上创建文件:
在ECS上运行 nano /home/chat-app/index.html 或 vim /home/chat-app/index.html。
将修改后的HTML代码粘贴进去并保存。
（可选）在ECS上简单托管前端文件:
您可以在ECS上快速启动一个简单的HTTP服务器来托管 index.html。
安装 http-server:
npm install -g http-server
在 /home/chat-app 目录下运行:
http-server -p 8000
现在您可以通过 http://<YOUR_ECS_PUBLIC_IP>:8000 访问您的聊天页面。
# 第五步：测试
确保服务器运行: 确保您的Node.js WebSocket服务器 (server.js) 正在ECS上运行。
访问前端页面:
本地测试: 在您的本地计算机上，用浏览器打开您修改后的 index.html 文件（通过文件浏览器打开，URL是 file://...）。由于是 file:// 协议，可以连接到 ws://。
ECS托管测试: 如果您在ECS上启动了 http-server，则在浏览器中访问 http://<YOUR_ECS_PUBLIC_IP>:8000。
进行聊天: 在两个不同的浏览器标签页或不同的设备上打开前端页面。
输入信息: 在每个页面上输入昵称、选择性别、设置距离，然后点击“开始聊天”。
验证连接: 查看页面状态是否显示“已连接到服务器”，并在服务器终端查看是否有连接日志。
验证消息: 在一个页面发送消息，检查另一个页面是否能收到（且符合性别和距离匹配规则）。
这样，您就完成了将整个项目从代码到部署到阿里云ECS的流程。记得在完成后关闭或释放ECS实例以避免不必要的费用。
