+++
date = '2025-05-26T11:48:25+08:00'
draft = false
title = 'Wireshark插件开发实战-Lua解析自定义二进制协议'
categories = ["网络抓包", "协议解析", "开发实战"]
tags = ["lua", "wireshark", "抓包分析", "网络调试", "插件开发", "tcpdump"]

+++

# Wireshark插件开发实战-Lua解析自定义二进制协议

在调试自定义 TCP 协议时，我们常常需要构造原始的二进制数据包并通过 Wireshark 进行抓包分析。这篇文章记录使用lua开发Wireshark插件解析私有二进制协议。
- 使用 **Python 构造二进制协议数据包**
- 使用 **`nc`（netcat）发送数据**
- 通过 **tcpdump** 抓包
- 通过 **Wireshark** + lua插件解析数据包

---

## 🧩 协议结构

协议格式如下：

```
[ uint16: msgType ][ uint16: bodyLen ][ bytes[bodyLen]: JSON字符串 ]
```

例如：

* msgType = `1`（0x0001）
* bodyLen = `9`（0x0009）
* body = `{"a":123}`

---

## 🛠️ 步骤 1：使用 Python 构造二进制数据

使用 Python 将结构化数据写入一个二进制文件 `msg.bin`：

```bash
python3 -c 'with open("msg.bin", "wb") as f: f.write((1).to_bytes(2, "big") + len(b"{\"a\":123}").to_bytes(2, "big") + b"{\"a\":123}")'
```
也可以使用xxd
```bash

echo '000100097b2261223a3132337d' | xxd -r -p > msg.bin

```

文件内容即为：

* `00 01`: msgType = 1
* `00 09`: bodyLen = 9
* `7b 22 61 22 3a 31 32 33 7d`: JSON = `{"a":123}`

可以使用 `xxd` 验证内容：

```bash
xxd msg.bin
```

![消息包](/note/images/mgs-bin.png)

---

## 🚀 步骤 2：使用 `nc` 模拟客户端向服务器发送数据

通过nc模拟tcp服务器监听端口，例如 `12345`：

```bash
nc -l -p 12345
```

通过nc模拟tcp客户端，往服务端发送消息：

```bash
# 发送数据到监听端口 12345
nc 127.0.0.1 12345 < msg.bin
# 也可以通过 xxd生成二进制数据直接发送
echo '0001 0009 7b2261223a3132337d' | xxd -r -p | nc 127.0.0.1 12345
```

---

## 📱 步骤 3：使用 tcpdump 抓包

sudo tcpdump -i lo port 12345 -w capture.pcap

1. 启动 Wireshark 打开

![加载插件前](/note/images/without-lua-plugin.png)

2. 基于lua开发wireshark插件

```lua
--[[ 
   sample.lua © 2025 xinchen@af83f787e8911dea9b3bf677746ebac9

   A simple Wireshark Lua dissector for a custom protocol whose payload is:
     [ uint16 msgType ][ uint16 bodyLen ][ bytes[bodyLen] containing a JSON string ]

   To use:
   1. Save this file as “sample.lua” (or any name ending in .lua).
   2. Place it into your Wireshark plugins directory (e.g., on Windows: 
      C:\Program Files\Wireshark\plugins\<version>\ or under ~/.config/wireshark/plugins/ for Linux/Mac).
   3. Restart Wireshark.
   4. If your protocol runs over TCP on port 12345, it will now decode automatically.
      (Adjust the port number in the last lines below to match your actual port.)
--]]

-- 1. Define the protocol
local sample = Proto("sample", "My JSON‐Over‐Binary Protocol")

-- 2. Define the fields we want to extract:
--    - msgType:   uint16
--    - bodyLen:   uint16
--    - jsonStr:   the JSON text (bytes interpreted as string)
local f_msgType = ProtoField.uint16("sample.msgType", "Message Type", base.DEC)
local f_bodyLen = ProtoField.uint16("sample.bodyLen", "Body Length", base.DEC)
local f_jsonStr = ProtoField.string("sample.json", "JSON Payload", base.ASCII)

sample.fields = { f_msgType, f_bodyLen, f_jsonStr }

-- 3. The dissector function
function sample.dissector(buffer, pinfo, tree)
    -- buffer:            the entire packet’s raw bytes
    -- pinfo:             packet metadata (e.g. columns, protocol)
    -- tree:              the protocol tree to which we add our parsed fields

    -- First, ensure we have at least 4 bytes for msgType + bodyLen
    if buffer:len() < 4 then
        return 0
    end

    -- Tell Wireshark which protocol column to display
    pinfo.cols.protocol = sample.name

    -- Add a subtree “My JSON‐Over‐Binary Protocol”
    local subtree = tree:add(sample, buffer(), "My JSON‐Over‐Binary Protocol Data")

    local offset = 0

    -- 3.1 Parse msgType (2 bytes)
    local msgType_field = buffer(offset, 2)
    local msgType_val = msgType_field:uint()
    subtree:add(f_msgType, msgType_field)
    offset = offset + 2

    -- 3.2 Parse bodyLen (2 bytes)
    local bodyLen_field = buffer(offset, 2)
    local bodyLen_val = bodyLen_field:uint()
    subtree:add(f_bodyLen, bodyLen_field)
    offset = offset + 2

    -- 3.3 Check if full JSON payload is available
    if buffer:len() < offset + bodyLen_val then
        -- If not enough bytes, mark it as malformed/truncated
        subtree:add_expert_info(PI_MALFORMED, PI_ERROR, "Packet too short for advertised bodyLen")
        return
    end

    -- 3.4 Extract the JSON string bytes, interpret as ASCII/UTF-8
    local json_field = buffer(offset, bodyLen_val)
    -- We use :string() so that Wireshark will display it as text
    subtree:add(f_jsonStr, json_field)
    offset = offset + bodyLen_val

    -- (Optional) If you want to pretty-print or validate JSON,
    -- you could attempt to parse it here or use a heuristic.
    -- But for most purposes, displaying the raw string is enough.
end

-- 4. Register the dissector on a specific TCP port (e.g. 12345).
--    Change “12345” to whatever port your protocol actually uses, or
--    alternatively hook into a heuristic/UDP/etc. as desired.
local tcp_port = DissectorTable.get("tcp.port")
tcp_port:add(12345, sample)

-- If your protocol is over UDP, swap the last two lines with:
-- local udp_port = DissectorTable.get("udp.port")
-- udp_port:add(12345, sample)

```

保存为 `sample.lua` 并放入 Wireshark 插件目录中（如 `~/.config/wireshark/plugins/`，或window中` C:\Program Files\Wireshark\plugins\<version>\`），重新打开 Wireshark 即可。

![加载插件后](/note/images/with-lua-plugin.png)



## 🥪 步骤 4（可选）：使用 `tcpdump` 抓包到 `.pcap` 文件

如果你希望在终端中进行抓包或生成 `.pcap` 文件用于后续分析，可以在 **Windows 主机上安装 WinDump（tcpdump 的 Windows 版本）**：

1. 安装 [Npcap](https://nmap.org/npcap/)

2. 安装 [WinDump](https://www.winpcap.org/windump/)

3. 查看接口列表：

   ```bash
   windump -D
   ```

4. 选择正确接口编号，开始抓包：

   ```bash
   windump -i 3 tcp port 12345 -w test.pcap
   ```

5. 使用 Wireshark 打开 `test.pcap` 文件进行分析。

---

## ✅ 总结

通过以上方式，我们完成了以下目标：

* ✅ 构造自定义协议的二进制数据
* ✅ 使用 `nc` 模拟客户端及服务端发送数据
* ✅ 使用 Wireshark 和 tcpdump 抓包
* ✅ 编写 Lua 脚本进行协议解析

这种方法适合调试任何基于 TCP/UDP 的自定义协议，非常适合开发和测试场景。

---

如果你也在调试自己的私有协议，不妨用这个流程走一遍！希望对你有帮助 😎
