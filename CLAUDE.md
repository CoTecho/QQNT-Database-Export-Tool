# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

QQNT 聊天记录导出工具 - 用于从已解密的 QQNT 数据库中导出聊天记录为 JSON/CSV/TXT 格式。

## 常用命令

```bash
# 安装依赖 (使用 uv 包管理器)
pip install uv --index-url=https://pypi.org/simple/ --trusted-host pypi.org
uv sync

# 激活虚拟环境
.venv\Scripts\activate

# 运行主程序导出聊天记录
python main.py --format json   # 导出为 JSON (默认)
python main.py --format csv    # 导出为 CSV
python main.py --format txt    # 导出为纯文本

# 代码检查
pylint db/ nt_msg/
```

## 架构概述

### 核心模块

1. **db/** - 数据库访问层
   - `man.py`: `DatabaseManager` 单例，管理 SQLAlchemy 会话，使用装饰器 `@DatabaseManager.register_model()` 注册模型
   - `models.py`: SQLAlchemy ORM 模型，映射 QQNT 数据库表结构（数字编码列名如 `40001`）

2. **nt_msg/** - 消息解析层
   - `message.py`: 消息解析核心
     - `Message` 类：从数据库对象解析消息
     - `ElementRegistry`: 元素类型注册表，使用 `@ElementRegistry.register(elem_id=N)` 注册解码器
     - `Element` 子类：Text, Image, Audio, Video, Emoji, Reply, App, Sticker 等消息元素类型
   - `contact.py`: 联系人/群组抽象类（Contact, Group）

### 数据流

```
nt_msg.decrypt.db → DatabaseManager → GroupMessage/PrivateMessage ORM
                                           ↓
                                    blackboxprotobuf.decode_message()
                                           ↓
                                    ElementRegistry.decode()
                                           ↓
                                    Message.elements → 导出文件
```

### 关键依赖

- `blackboxprotobuf`: 解析 protobuf 格式的消息体
- `sqlalchemy`: ORM 数据库访问
- `pydantic`: 数据模型验证

## 数据库约定

- 数据库文件命名：`{db_name}.decrypt.db`（如 `nt_msg.decrypt.db`）
- 列名使用数字编码（如 `40001` 对应 `msgId`）
- 消息体 `msgBody` 字段为 protobuf 编码的二进制数据

## 输出目录

- `chathistory/`: 导出的聊天记录存放目录
- `group_chat_history_{groupUin}.{format}`: 群聊记录
- `c2c_chat_history_{peerUin}.{format}`: 私聊记录

## 添加新消息元素类型

```python
@ElementRegistry.register(elem_id=N)  # N 对应 protobuf 中的元素类型 ID
class NewElement(Element):
    # 属性定义...

    @classmethod
    def decode(cls, data):
        # 从 protobuf 解码数据
        return NewElement(...)

    def __str__(self):
        # 返回可读的字符串表示
        return "[新元素类型]"
```
