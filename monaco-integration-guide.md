# Cline + Monaco在线IDE集成方案

## 🎯 集成目标

将Cline AI编程助手集成到基于Monaco Editor的在线IDE中，提供智能代码生成、文件操作、终端执行等功能。

## 🏗️ 整体架构

```
Monaco在线IDE
├── Monaco Editor (代码编辑器)
├── File Explorer (文件浏览器)
├── Terminal Panel (终端面板)
├── Cline Chat Panel (AI助手面板)
└── Backend Services
    ├── Cline Standalone Server (AI服务)
    ├── File System API (文件操作)
    ├── Terminal Service (命令执行)
    └── WebSocket/gRPC (通信层)
```

## 🔧 技术实现方案

### 方案一：基于Cline Standalone模式 (推荐)

#### 1. 后端服务部署

```bash
# 1. 克隆Cline项目
git clone https://github.com/cline/cline.git
cd cline

# 2. 安装依赖并构建
npm install
npm run build

# 3. 启动standalone服务
npm run standalone
```

#### 2. 前端集成步骤

**2.1 安装Monaco Editor**
```bash
npm install monaco-editor
npm install @monaco-editor/react
```

**2.2 创建Cline客户端**
```typescript
// src/services/ClineClient.ts
import { WebSocket } from 'ws';

export class ClineClient {
  private ws: WebSocket;
  private messageHandlers: Map<string, Function> = new Map();

  constructor(serverUrl: string) {
    this.ws = new WebSocket(serverUrl);
    this.setupEventHandlers();
  }

  private setupEventHandlers() {
    this.ws.on('message', (data) => {
      const message = JSON.parse(data.toString());
      const handler = this.messageHandlers.get(message.type);
      if (handler) {
        handler(message.payload);
      }
    });
  }

  // 发送消息给Cline
  async sendMessage(type: string, payload: any) {
    const message = { type, payload, id: Date.now() };
    this.ws.send(JSON.stringify(message));
  }

  // 注册消息处理器
  onMessage(type: string, handler: Function) {
    this.messageHandlers.set(type, handler);
  }

  // AI对话
  async chat(message: string, files?: string[]) {
    return this.sendMessage('chat', { message, files });
  }

  // 文件操作
  async editFile(path: string, content: string) {
    return this.sendMessage('editFile', { path, content });
  }

  // 执行命令
  async runCommand(command: string) {
    return this.sendMessage('runCommand', { command });
  }
}
```

**2.3 Monaco编辑器集成**
```typescript
// src/components/MonacoEditor.tsx
import React, { useRef, useEffect } from 'react';
import { Editor } from '@monaco-editor/react';
import { ClineClient } from '../services/ClineClient';

interface MonacoEditorProps {
  clineClient: ClineClient;
}

export const MonacoEditor: React.FC<MonacoEditorProps> = ({ clineClient }) => {
  const editorRef = useRef<any>(null);

  useEffect(() => {
    // 监听Cline的文件编辑事件
    clineClient.onMessage('fileEdited', (data) => {
      const { path, content } = data;
      if (editorRef.current) {
        editorRef.current.setValue(content);
      }
    });

    // 注册右键菜单
    if (editorRef.current) {
      editorRef.current.addAction({
        id: 'cline-explain',
        label: 'Explain with Cline',
        contextMenuGroupId: 'cline',
        run: () => {
          const selection = editorRef.current.getSelection();
          const selectedText = editorRef.current.getModel().getValueInRange(selection);
          clineClient.chat(`请解释这段代码: ${selectedText}`);
        }
      });

      editorRef.current.addAction({
        id: 'cline-improve',
        label: 'Improve with Cline',
        contextMenuGroupId: 'cline',
        run: () => {
          const selection = editorRef.current.getSelection();
          const selectedText = editorRef.current.getModel().getValueInRange(selection);
          clineClient.chat(`请优化这段代码: ${selectedText}`);
        }
      });
    }
  }, [clineClient]);

  const handleEditorDidMount = (editor: any) => {
    editorRef.current = editor;
  };

  return (
    <Editor
      height="100vh"
      defaultLanguage="typescript"
      theme="vs-dark"
      onMount={handleEditorDidMount}
      options={{
        contextmenu: true,
        minimap: { enabled: true },
        fontSize: 14,
        wordWrap: 'on',
      }}
    />
  );
};
```

**2.4 Cline聊天面板**
```typescript
// src/components/ClineChat.tsx
import React, { useState, useEffect } from 'react';
import { ClineClient } from '../services/ClineClient';

interface ClineChatProps {
  clineClient: ClineClient;
}

export const ClineChat: React.FC<ClineChatProps> = ({ clineClient }) => {
  const [messages, setMessages] = useState<any[]>([]);
  const [inputValue, setInputValue] = useState('');
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    // 监听Cline响应
    clineClient.onMessage('chatResponse', (response) => {
      setMessages(prev => [...prev, { type: 'assistant', content: response.content }]);
      setIsLoading(false);
    });

    // 监听任务状态
    clineClient.onMessage('taskStatus', (status) => {
      // 更新任务状态UI
      console.log('Task status:', status);
    });
  }, [clineClient]);

  const handleSendMessage = async () => {
    if (!inputValue.trim()) return;

    setMessages(prev => [...prev, { type: 'user', content: inputValue }]);
    setIsLoading(true);
    
    await clineClient.chat(inputValue);
    setInputValue('');
  };

  return (
    <div className="cline-chat-panel">
      <div className="chat-messages">
        {messages.map((msg, index) => (
          <div key={index} className={`message ${msg.type}`}>
            {msg.content}
          </div>
        ))}
        {isLoading && <div className="loading">Cline正在思考...</div>}
      </div>
      
      <div className="chat-input">
        <input
          type="text"
          value={inputValue}
          onChange={(e) => setInputValue(e.target.value)}
          onKeyPress={(e) => e.key === 'Enter' && handleSendMessage()}
          placeholder="告诉Cline你想要做什么..."
        />
        <button onClick={handleSendMessage}>发送</button>
      </div>
    </div>
  );
};
```

**2.5 主应用集成**
```typescript
// src/App.tsx
import React, { useEffect, useState } from 'react';
import { MonacoEditor } from './components/MonacoEditor';
import { ClineChat } from './components/ClineChat';
import { ClineClient } from './services/ClineClient';

export const App: React.FC = () => {
  const [clineClient, setClineClient] = useState<ClineClient | null>(null);

  useEffect(() => {
    // 初始化Cline客户端
    const client = new ClineClient('ws://localhost:8080'); // Cline服务地址
    setClineClient(client);

    return () => {
      // 清理连接
      client.disconnect?.();
    };
  }, []);

  if (!clineClient) {
    return <div>连接Cline服务中...</div>;
  }

  return (
    <div className="ide-container">
      <div className="editor-panel">
        <MonacoEditor clineClient={clineClient} />
      </div>
      <div className="sidebar">
        <ClineChat clineClient={clineClient} />
      </div>
    </div>
  );
};
```

### 方案二：直接集成Cline Core模块

#### 1. 提取核心模块
```bash
# 只安装Cline的核心依赖
npm install @anthropic-ai/sdk axios
```

#### 2. 适配器模式实现
```typescript
// src/adapters/ClineAdapter.ts
import { Controller } from 'cline/src/core/controller';
import { Task } from 'cline/src/core/task';

export class MonacoClineAdapter {
  private controller: Controller;
  
  constructor() {
    // 创建模拟的VSCode context
    const mockContext = this.createMockVSCodeContext();
    this.controller = new Controller(mockContext, console, this.postMessage.bind(this));
  }

  private createMockVSCodeContext() {
    // 模拟VSCode扩展上下文
    return {
      globalState: new Map(),
      workspaceState: new Map(),
      secrets: new Map(),
      // ... 其他必要的模拟实现
    };
  }

  private postMessage(message: any) {
    // 将Cline消息转发到Monaco IDE
    window.dispatchEvent(new CustomEvent('cline-message', { detail: message }));
  }

  // 包装Cline API
  async chat(message: string) {
    return this.controller.initTask(message);
  }

  async editFile(path: string, content: string) {
    // 调用Cline的文件编辑工具
    // ...
  }
}
```

## 🔌 API接口设计

### WebSocket消息格式
```typescript
interface ClineMessage {
  id: string;
  type: 'chat' | 'fileEdit' | 'command' | 'status';
  payload: any;
  timestamp: number;
}

// 用户消息
interface ChatMessage {
  type: 'chat';
  payload: {
    message: string;
    files?: string[];
    images?: string[];
  };
}

// 文件编辑
interface FileEditMessage {
  type: 'fileEdit';
  payload: {
    path: string;
    content: string;
    operation: 'create' | 'modify' | 'delete';
  };
}

// 命令执行
interface CommandMessage {
  type: 'command';
  payload: {
    command: string;
    cwd?: string;
  };
}
```

### REST API设计
```typescript
// GET /api/files/:path - 获取文件内容
// POST /api/files/:path - 创建/更新文件
// DELETE /api/files/:path - 删除文件
// POST /api/chat - 发送聊天消息
// POST /api/commands - 执行命令
// GET /api/status - 获取Cline状态
```

## 🎨 UI/UX集成建议

### 1. 聊天面板设计
- 侧边栏或底部面板
- 支持Markdown渲染
- 代码高亮显示
- 文件引用显示
- 任务进度指示器

### 2. 编辑器增强
- 右键菜单集成Cline功能
- 代码选择后的快速操作
- AI建议的内联显示
- 差异视图支持

### 3. 文件浏览器集成
- 显示AI修改的文件
- 文件变更历史
- 一键回滚功能

## 🔧 部署配置

### Docker部署
```dockerfile
# Dockerfile
FROM node:18-alpine

WORKDIR /app

# 复制Cline项目
COPY cline/ ./cline/
COPY monaco-ide/ ./monaco-ide/

# 安装依赖
RUN cd cline && npm install && npm run build
RUN cd monaco-ide && npm install && npm run build

# 启动服务
CMD ["npm", "run", "start:production"]
```

### 环境变量配置
```bash
# .env
CLINE_PORT=8080
MONACO_PORT=3000
API_PROVIDER=anthropic
ANTHROPIC_API_KEY=your_key_here
OPENAI_API_KEY=your_key_here
```

## 🚀 快速开始

1. **克隆项目**
```bash
git clone https://github.com/cline/cline.git
cd cline
```

2. **安装依赖**
```bash
npm install
```

3. **配置API密钥**
```bash
cp .env.example .env
# 编辑.env文件，添加你的API密钥
```

4. **启动开发服务**
```bash
# 启动Cline standalone服务
npm run standalone

# 启动Monaco IDE
npm run dev:monaco
```

5. **访问应用**
```
http://localhost:3000
```

## 📝 注意事项

1. **安全性**：确保API密钥安全，使用环境变量
2. **性能**：大文件操作时注意内存使用
3. **兼容性**：测试不同浏览器的兼容性
4. **错误处理**：完善的错误处理和用户反馈
5. **权限控制**：文件系统访问权限管理

## 🔄 后续扩展

1. **多用户支持**：工作区隔离
2. **插件系统**：自定义工具扩展
3. **云端同步**：项目云端存储
4. **协作功能**：多人实时协作
5. **AI模型切换**：支持更多AI提供商

这个方案提供了完整的集成路径，你可以根据具体需求选择合适的实现方式。 