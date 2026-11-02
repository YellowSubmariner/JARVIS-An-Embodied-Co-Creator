# Kinova Gen3：新电脑从零配置 Kortex API 环境与 J2/J3 标定

整理日期：2026-11-02。面向 Windows 电脑，按本次实际连接、写入及回读记录整理。

本文先把电脑配置到能够通过 Python Kortex API 登录、读取机械臂的状态，再说明这台机器 J2/J3 的手动摆位、角度参考标定和 Codex 提示词。环境安装完成后，日常连接不需要重复标定。

## 1. 先明确这台机器的情况

**本台机器的 J2、J3 是坏的，具体表现为磁编码器与读数编码器存在偏差，角度参考异常并报警。这是操作者在现场已经得出的结论。操作者此前已经标定成功，并用标定后的机械臂成功进行过 VLA 推理。**

本次处理顺序是：

1. 先配置电脑，让它能通过网线访问机械臂的 Kortex API。
2. **现场手动把机械臂摆到已知正确的位置。**
3. 在该物理姿态下，将第二关节 J2 的当前角度登记为 **340°**，将第三关节 J3 的当前角度登记为 **180°**。
4. 通过 API 回读角度、偏移量及故障状态，确认写入结果。

这里的 340°、180°是给当前实际姿态登记的角度参考。操作者已经摆好位置后，调用 `SetAxisOffsets`；不需要再发送运动命令把关节转到这些数字。

这两个数值属于本台机器、本次已知物理姿态的定义。换一台机械臂或换一个物理姿态，要重新确认目标值。

## 2. 本次已经实际跑通的配置

| 项目 | 本次配置 |
| --- | --- |
| 机械臂 | Kinova Gen3 Ultra Lightweight Robot，7 自由度 |
| 基座型号 | L53K |
| 机械臂 IP | `192.168.1.10` |
| 网页地址 | `http://192.168.1.10/monitoring` |
| Python API 连接 | TCP，端口 `10000` |
| Python | Windows 64 位 CPython `3.13.7` |
| Kortex SDK | `kortex_api-2.8.0.post5-py3-none-any.whl` |
| protobuf | `3.20.0` |
| Deprecated | `1.2.7` |
| wrapt | `1.17.3` |
| 本次登录用户名 | `admin`，密码在运行时输入 |

这是本机实测成功的组合；不是对所有机械臂型号和 Python 版本的兼容性承诺。下面固定这些版本，方便复现。网页当时显示 `2.9.0-r.15`，这个网页版本号不要直接当成 Python SDK 包版本或基座组件的版本字符串。

Kinova 的 [官方下载说明](https://github.com/Kinovarobotics/Kinova-kortex2_Gen3_G3L/blob/master/readme.md)提供 Gen3 SDK 下载入口；官方 [Python 示例说明](https://github.com/Kinovarobotics/Kinova-kortex2_Gen3_G3L/blob/master/api_python/examples/readme.md)采用 wheel 安装方式。

## 3. 准备电脑、网线和工作目录

需要准备 Windows 电脑、可用的有线网口或 USB 网卡、网线，以及机械臂电源。下载 Python 和 SDK 时电脑需要联网；直接连接机械臂的网线可以与用于下载的 Wi-Fi 分别使用。

在 Windows 开始菜单中打开 **PowerShell**。下文标为 `powershell` 的代码都在 PowerShell 执行；标为 `python` 的代码保存成 `.py` 文件后运行。

建立一个固定的项目目录：

```powershell
$KortexProject = Join-Path $env:USERPROFILE 'Documents\KinovaKortex'
New-Item -ItemType Directory -Path $KortexProject -Force | Out-Null
Set-Location -LiteralPath $KortexProject
New-Item -ItemType Directory -Path 'work','scripts','outputs' -Force | Out-Null
Get-Location
```

后续命令都从这个项目目录执行。目录用途如下：

```text
KinovaKortex/
  work/                         SDK 下载文件、虚拟环境
    kortex-env/
  scripts/                      Python 脚本
    download_sdk.py
    inspect_kortex.py
  outputs/                      每次检查及标定的 JSON 记录
```

如果关闭了 PowerShell，下次先重新进入自己的 `Documents\KinovaKortex` 目录。

## 4. 接通机械臂网络

### 4.1 先检查是否已经能连接

电脑用网线连接机械臂，机械臂上电。在 PowerShell 执行：

```powershell
ipconfig
Test-NetConnection 192.168.1.10 -Port 10000
```

重点看：

```text
TcpTestSucceeded : True
```

如果已经为 `True`，直接进入 Python 配置，不必改现有网络设置。网页也可以辅助查看连接情况，但浏览器能打开网页并不等于 Python API 已登录成功。

### 4.2 不能连接时，检查机械臂有线网卡的 IPv4

Kinova 官方示例使用同网段静态地址，例如电脑 `192.168.1.11/24`、机械臂 `192.168.1.10`。参见 [官方网络准备说明](https://github.com/Kinovarobotics/Kinova-kortex2_Gen3_G3L/blob/master/api_python/examples/readme.md)。

在 Windows 的“设置 → 网络和 Internet → 以太网”中，选中**连接机械臂的那块网卡**，将 IP 分配编辑为手动 IPv4：

| 设置项 | 直连本台机械臂的示例值 |
| --- | --- |
| 电脑 IPv4 地址 | `192.168.1.11`，需确认没有被其他设备占用 |
| 子网掩码 | `255.255.255.0` |
| 若界面要求前缀长度 | `24` |
| 默认网关 | 仅直连机械臂时留空 |
| DNS | 仅通过 IP 访问机械臂时不需要填写 |

电脑地址不能也填成 `192.168.1.10`。不要把用于上网的另一块网卡改成上述地址。Windows 的手动 IPv4 设置入口可参见 [微软官方说明](https://support.microsoft.com/en-gb/windows/experience/connectivity-networking/essential-network-settings-and-tasks-in-windows)。

改好后重新执行 `Test-NetConnection`。如果仍然失败，按顺序检查网线、网卡、机械臂电源、机械臂实际 IP，以及该网段是否被其他网络接口或路由占用。

本次电脑原先已经能访问机械臂，没有实际修改其静态 IP；本节提供新电脑首次连接时的配置方法。

## 5. 安装并确认 Python

为复现本次环境，可从 [Python 3.13.7 官方发布页](https://www.python.org/downloads/release/python-3137/)下载 **Windows installer (64-bit)**。安装时选择普通 CPython，并安装 pip；可勾选将 Python 加入 PATH。完成后重新打开 PowerShell。

如果电脑已有合适的 Python，先检查，不必重复安装：

```powershell
python --version
python -c "import sys; print(sys.executable); print(sys.version)"
python -m pip --version
```

本次实际使用的是 Python `3.13.7`。其他版本需要重新完成导入和只读连接验收。

如果 `python` 无法运行或指向了其他解释器，先查路径：

```powershell
Get-Command python -ErrorAction SilentlyContinue
```

普通用户安装的常见路径示例为：

```powershell
& "$env:LOCALAPPDATA\Programs\Python\Python313\python.exe" --version
```

实际路径以本机安装位置为准。本次 `py -0p` 曾返回找不到已安装 Python，但直接调用 `python.exe` 正常；不能仅凭 `py` 启动器的结果认定电脑没有 Python。

## 6. 创建独立虚拟环境

回到项目目录，执行：

```powershell
python -m venv .\work\kortex-env
.\work\kortex-env\Scripts\python.exe --version
.\work\kortex-env\Scripts\python.exe -m pip --version
```

如果前一步使用的是 Python 安装目录的完整路径，这里的创建命令也使用同一个解释器：

```powershell
& "$env:LOCALAPPDATA\Programs\Python\Python313\python.exe" -m venv .\work\kortex-env
```

创建成功后，后面的安装和运行命令全部调用 `work\kortex-env\Scripts\python.exe`。不需要激活环境，也不需要为激活脚本修改 PowerShell 执行策略。虚拟环境用法参见 [Python 官方说明](https://docs.python.org/3.13/tutorial/venv.html)。

## 7. 下载并校验 Kortex SDK

本次使用的官方 wheel：

[kortex_api-2.8.0.post5-py3-none-any.whl](https://artifactory.kinovaapps.com/artifactory/generic-local-public/kortex/API/2.8.0/kortex_api-2.8.0.post5-py3-none-any.whl)

本次下载文件与 Kinova Artifactory 返回的 SHA256 一致：

```text
5796425d48f0ab70c56ce9ecf06b8ab91f0b951319266e5d4c0b179cdb9e99cf
```

用记事本新建 `scripts\download_sdk.py`，保存为 UTF-8。保存类型选择“所有文件”，确认文件名没有变成 `.py.txt`。复制以下完整内容：

```python
from pathlib import Path
import hashlib
import urllib.request

url = (
    "https://artifactory.kinovaapps.com/artifactory/"
    "generic-local-public/kortex/API/2.8.0/"
    "kortex_api-2.8.0.post5-py3-none-any.whl"
)
expected = "5796425d48f0ab70c56ce9ecf06b8ab91f0b951319266e5d4c0b179cdb9e99cf"
target = Path("work/kortex_api-2.8.0.post5-py3-none-any.whl")
data = urllib.request.urlopen(url, timeout=60).read()
actual = hashlib.sha256(data).hexdigest()
if actual != expected:
    raise RuntimeError(f"SHA256 mismatch: {actual}")
target.parent.mkdir(parents=True, exist_ok=True)
target.write_bytes(data)
print("Saved:", target)
print("SHA256:", actual)
```

运行：

```powershell
.\work\kortex-env\Scripts\python.exe .\scripts\download_sdk.py
```

也可以通过浏览器从上述官方下载链接保存到 `work`，然后校验：

```powershell
Get-FileHash -LiteralPath '.\work\kortex_api-2.8.0.post5-py3-none-any.whl' -Algorithm SHA256
```

本次 PowerShell 的部分 HTTPS 下载请求报过认证错误，而 Python `urllib` 下载成功。遇到这类差异，可使用上面的 Python 下载方式；不要通过关闭 TLS 证书验证来解决。

## 8. 安装固定版本依赖和 SDK

```powershell
.\work\kortex-env\Scripts\python.exe -m pip install --no-cache-dir "protobuf==3.20.0" "Deprecated==1.2.7" "wrapt==1.17.3"
.\work\kortex-env\Scripts\python.exe -m pip install --no-cache-dir --no-deps .\work\kortex_api-2.8.0.post5-py3-none-any.whl
.\work\kortex-env\Scripts\python.exe -m pip check
```

`--no-deps` 用在第二条，是因为依赖已经按本次成功组合单独安装。此 wheel 的元数据要求 protobuf `>=3.5.1, <=3.20` 和 Deprecated `==1.2.7`；不要将 protobuf 单独升级到新大版本。

检查导入：

```powershell
.\work\kortex-env\Scripts\python.exe -c "from kortex_api.TCPTransport import TCPTransport; from kortex_api.autogen.client_stubs.BaseClientRpc import BaseClient; from kortex_api.autogen.client_stubs.ActuatorConfigClientRpc import ActuatorConfigClient; print('Kortex imports OK')"
.\work\kortex-env\Scripts\python.exe -m pip list
```

预期看到 `Kortex imports OK`，以及第 2 节中的四个已安装包版本。到这里，SDK 软件环境已经具备。

## 9. 建立只读登录与检查脚本

将以下完整内容保存为 `scripts\inspect_kortex.py`。脚本创建临时 API 会话，读取基座、关节角度、编码器偏移和故障信息，然后关闭会话；不会调用标定、清故障、运动、使能、模式切换或重启命令。

密码通过终端隐藏输入，不写入脚本或 JSON。用户名默认 `admin`；如果设备使用其他账户，通过 `--username` 指定。

```python
"""Read robot state through the official Kortex SDK; no robot-setting writes.

Only CreateSession/KeepAlive/CloseSession alter the temporary API session.
This snapshot is diagnostic evidence, not a complete restorable calibration.
"""

import argparse
import getpass
import json
import os
from datetime import datetime, timezone, timedelta
from pathlib import Path

from google.protobuf.json_format import MessageToDict
from kortex_api.TCPTransport import TCPTransport
from kortex_api.RouterClient import RouterClient, RouterClientSendOptions
from kortex_api.SessionManager import SessionManager
from kortex_api.autogen.client_stubs.ActuatorConfigClientRpc import ActuatorConfigClient
from kortex_api.autogen.client_stubs.BaseClientRpc import BaseClient
from kortex_api.autogen.client_stubs.BaseCyclicClientRpc import BaseCyclicClient
from kortex_api.autogen.client_stubs.DeviceConfigClientRpc import DeviceConfigClient
from kortex_api.autogen.client_stubs.DeviceManagerClientRpc import DeviceManagerClient
from kortex_api.autogen.messages import Session_pb2, Common_pb2, ActuatorConfig_pb2


def as_dict(message):
    return MessageToDict(message, preserving_proto_field_name=True,
                         including_default_value_fields=True)


def safety_names(mask):
    names = []
    known = 0
    for name, value in ActuatorConfig_pb2.SafetyIdentifierBankA.items():
        if value and (mask & value) == value:
            names.append(name)
            known |= value
    if mask & ~known:
        names.append(f"UNKNOWN_BITS_0x{mask & ~known:X}")
    return names


def main():
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("--ip", default="192.168.1.10")
    parser.add_argument("--username", default="admin")
    parser.add_argument("--output", type=Path, required=True)
    args = parser.parse_args()
    password = os.environ.pop("KORTEX_PASSWORD", None)
    if password is None:
        password = getpass.getpass("Kortex password: ")

    report = {
        "captured_at": datetime.now(timezone(timedelta(hours=8))).isoformat(),
        "target": args.ip,
        "sdk_version": "2.8.0.post5",
        "operation": "read_only_diagnostic_snapshot",
        "this_script_wrote_calibration": False,
        "errors": [],
    }
    options = RouterClientSendOptions()
    options.timeout_ms = 2500
    transport = TCPTransport()
    router = RouterClient(transport, RouterClient.basicErrorCallback)
    manager = SessionManager(router)
    connected = False
    session_created = False

    def read(label, call):
        try:
            return as_dict(call())
        except Exception as error:
            report["errors"].append({"read": label, "error": str(error)})
            return None

    try:
        transport.sock.settimeout(5)
        transport.connect(args.ip, 10000)
        transport.sock.settimeout(None)
        connected = True
        session = Session_pb2.CreateSessionInfo()
        session.username = args.username
        session.password = password
        session.session_inactivity_timeout = 10000
        session.connection_inactivity_timeout = 2000
        manager.CreateSession(session, options=options)
        session_created = True
        session.ClearField("password")
        del password

        base = BaseClient(router)
        cyclic = BaseCyclicClient(router)
        config = DeviceConfigClient(router)
        devices = DeviceManagerClient(router)
        actuator = ActuatorConfigClient(router)

        report["base_firmware"] = read("base.GetFirmwareVersion", lambda:
            config.GetFirmwareVersion(options=options))
        report["base_model"] = read("base.GetModelNumber", lambda:
            config.GetModelNumber(options=options))
        report["product_configuration"] = read("base.GetProductConfiguration", lambda:
            base.GetProductConfiguration(options=options))
        report["arm_state"] = read("base.GetArmState", lambda:
            base.GetArmState(options=options))
        report["operating_mode"] = read("base.GetOperatingMode", lambda:
            base.GetOperatingMode(options=options))
        report["servoing_mode"] = read("base.GetServoingMode", lambda:
            base.GetServoingMode(options=options))
        report["joint_angles"] = read("base.GetMeasuredJointAngles", lambda:
            base.GetMeasuredJointAngles(options=options))
        report["feedback"] = read("cyclic.RefreshFeedback", lambda:
            cyclic.RefreshFeedback(options=options))

        handles = devices.ReadAllDevices(options=options)
        report["device_handles"] = as_dict(handles)
        actuator_types = {Common_pb2.BIG_ACTUATOR, Common_pb2.SMALL_ACTUATOR,
                          Common_pb2.MEDIUM_ACTUATOR, Common_pb2.XBIG_ACTUATOR}
        report["actuators"] = []
        for handle in sorted(handles.device_handle, key=lambda h: h.order):
            if handle.device_type not in actuator_types:
                continue
            device_id = handle.device_identifier
            entry = {"device_id": device_id, "order": handle.order,
                     "device_type": Common_pb2.DeviceTypes.Name(handle.device_type)}
            reads = {
                "firmware": config.GetFirmwareVersion,
                "axis_offsets": actuator.GetAxisOffsets,
                "servoing": actuator.GetServoing,
                "command_mode": actuator.GetCommandMode,
                "control_mode": actuator.GetControlMode,
            }
            for field, method in reads.items():
                entry[field] = read(f"device_{device_id}.{method.__name__}",
                    lambda m=method, d=device_id: m(deviceId=d, options=options))
            report["actuators"].append(entry)

        feedback = report.get("feedback") or {}
        report["joint_feedback_summary"] = []
        for index, item in enumerate(feedback.get("actuators", []), start=1):
            report["joint_feedback_summary"].append({
                "joint_number": index,
                "position_degrees": item.get("position"),
                "velocity_degrees_per_second": item.get("velocity"),
                "fault_bank_a": item.get("fault_bank_a"),
                "fault_bank_b": item.get("fault_bank_b"),
                "warning_bank_a": item.get("warning_bank_a"),
                "warning_bank_b": item.get("warning_bank_b"),
                "warning_names_bank_a": safety_names(int(item.get("warning_bank_a", 0))),
                "fault_names_bank_a": safety_names(int(item.get("fault_bank_a", 0))),
            })
    except Exception as error:
        report["errors"].append({"stage": "connection_or_inspection", "error": str(error)})
    finally:
        if session_created:
            try:
                manager.CloseSession(options=options)
            except Exception as error:
                report["errors"].append({"cleanup": "CloseSession", "error": str(error)})
        if connected:
            try:
                transport.disconnect()
            except Exception as error:
                report["errors"].append({"cleanup": "disconnect", "error": str(error)})
        args.output.parent.mkdir(parents=True, exist_ok=True)
        args.output.write_text(json.dumps(report, ensure_ascii=False, indent=2),
                               encoding="utf-8")

    print(json.dumps({key: report.get(key) for key in (
        "captured_at", "base_firmware", "product_configuration", "arm_state", "operating_mode",
        "servoing_mode", "actuators", "joint_feedback_summary", "errors",
        "this_script_wrote_calibration")}, ensure_ascii=False, indent=2))

    return 1 if report["errors"] else 0


if __name__ == "__main__":
    raise SystemExit(main())
```

运行检查，输出文件名带时间，保留每次记录：

```powershell
$KortexStamp = Get-Date -Format 'yyyyMMdd_HHmmss'
.\work\kortex-env\Scripts\python.exe .\scripts\inspect_kortex.py --ip 192.168.1.10 --username admin --output ".\outputs\inspect_$KortexStamp.json"
```

出现 `Kortex password:` 时输入机械臂当前密码，再按 Enter。这里输入的是机械臂账户密码。

正常执行后应得到：

- `base_firmware`、`base_model`：从 API 实际读取的设备信息。
- `device_handles`：设备编号及其顺序。
- `joint_angles`、`joint_feedback_summary`：角度、速度、故障和警告。
- `actuators` 中的 `axis_offsets`、`servoing`：偏移量和伺服状态。
- `errors`：各读取调用的错误；成功时为 `[]`。

`this_script_wrote_calibration: false` 只表示这次检查脚本没有写标定，不表示机械臂没有标定。

JSON 是状态记录，不是完整、可直接恢复的出厂标定备份。

## 10. 怎样确认“这台新电脑配置好了”

| 层次 | 通过条件 | 未通过时主要检查 |
| --- | --- | --- |
| 网络 | TCP `10000` 可达 | 网线、IP、子网、路由 |
| Python 软件 | SDK 导入成功，`pip check` 通过 | 实际解释器、依赖版本 |
| API 会话 | 账户登录成功，能读到设备列表和角度 | 密码、设备地址、SDK 服务调用 |
| 机械臂运行状态 | 角度参考正确，故障/警告符合现场要求 | 机械臂状态、现场标定或恢复流程 |

**前三级通过，就说明电脑已经具备可使用 Kortex API 的环境。** 机械臂自身有编码器报警时，仍可能正常登录和读取；不要把机械臂报警误判为电脑安装失败。

这套环境可以作为后续控制程序或 VLA 程序调用机械臂 API 的基础。VLA 模型、相机和推理依赖需要在相应项目中另行配置。

## 11. J2/J3 的标定：先手动摆位置，再登记角度

### 11.1 现场准备

本台 J2/J3 的异常已经由现场操作者确认。操作者应使用已知姿态、夹具或已有现场方法，**手动摆好实际位置**，使该物理姿态对应 J2＝340°、J3＝180°。由于角度基准本身有偏差，摆位不能只依赖异常的 API 显示值。

摆好后确认：

1. 当前实际姿态就是要登记的姿态。
2. 已有独立机械支撑，关节失去保持力后也不会坠落或转动。
3. 外部运动/VLA 控制程序已经停止发送动作。
4. 先保存一份第 9 节的只读检查记录。

本次操作者还明确确认：此前在同一台机器、同一固件、伺服开启的状态下，直接调用 `SetAxisOffsets` 成功过。本次沿用了这个现场已验证的方式；不要把这个工作条件自动套用到其他设备。

### 11.2 确认关节编号

本次通过 `DeviceManagerClient.ReadAllDevices()` 实际确认：

| 物理关节 | ActuatorConfig 路由参数 | GetMeasuredJointAngles 返回的编号 |
| --- | --- | --- |
| J2，第二关节 | `deviceId=2`，设备 `order=2` | `joint_identifier=1` |
| J3，第三关节 | `deviceId=3`，设备 `order=3` | `joint_identifier=2` |

设备列表不是固定排序的。新程序应按设备句柄的 `device_identifier` 和 `order` 确认映射，不要把“列表中的第三项”直接当成 J3。参见 [官方设备路由说明](https://github.com/Kinovarobotics/Kinova-kortex2_Gen3_G3L/blob/master/linked_md/python_device_routing.md)。

### 11.3 关键 API 调用

准确的方法名是 **`ActuatorConfigClient.SetAxisOffsets`**。下面是已经登录的 `router` 会话内的核心代码，`AxisPosition.position` 的单位是度：

```python
from kortex_api.autogen.client_stubs.ActuatorConfigClientRpc import ActuatorConfigClient
from kortex_api.autogen.messages import ActuatorConfig_pb2

actuator = ActuatorConfigClient(router)

# 现场确认 J2 已摆好，对应实际角度 340°。
j2_reference = ActuatorConfig_pb2.AxisPosition()
j2_reference.position = 340.0
actuator.SetAxisOffsets(j2_reference, deviceId=2)
```

**执行一次后先记录确认响应、偏移量、角度和保护状态，再决定是否继续。** 如果发生保护故障，停止自动连续写入并核对现场状态；不要把写入超时当成“肯定没写入”而直接重试。

确认 J3 的现场姿态和继续操作条件后，其核心调用为：

```python
j3_reference = ActuatorConfig_pb2.AxisPosition()
j3_reference.position = 180.0
actuator.SetAxisOffsets(j3_reference, deviceId=3)
```

这两段是接口说明，必须放在已登录、且有可靠记录和退出处理的脚本会话中使用；不是可直接单独运行的完整脚本。第 12 节提供给 Codex 的执行提示词，由 Codex 在已配置环境里编写并执行完整脚本。

`GetAxisOffsets` 返回的是绝对/相对编码器偏移量；`SetAxisOffsets` 接受的是 `AxisPosition`。不要把回读的 `absolute_offset`、`relative_offset` 原样当成两个目标角度填进去，也不要把 `SetTorqueOffset` 扭矩零偏当成角度标定。

Kinova [官方 API 定义](https://github.com/Kinovarobotics/ros_kortex/blob/noetic-devel/kortex_driver/protos/ActuatorConfig.proto)说明 `SetAxisOffsets` 写入编码器偏移，并会使已有机械臂标定失效，同时标为 Kinova 专用。本节记录的是操作者明确要求、并已在本台机器上实际执行的角度参考登记流程。

### 11.4 本次实际发生的保护和最终验收

本次 J2、J3 的 `SetAxisOffsets` 均收到控制器确认。两次写入后，目标关节都曾出现 `MAXIMUM_VELOCITY`、`HALL_POSITION` 故障，并自动关闭对应关节伺服。脚本没有自动清故障、重新使能、切换控制模式或重启。

J2 写入后先暂停了 J3；操作者确认姿态没变、已独立支撑固定，并明确要求继续 J3 且先不要清故障或重新使能后，才执行 J3 写入。

**角度参考写入成功**与**机械臂恢复运行就绪**是两项检查。现场之后如何完成恢复，不是本次脚本执行的操作，本文不把它补写成一个未经记录的一键恢复步骤。

在 2026-11-02 18:16:14（北京时间）的独立只读检查中，得到：

| 关节 | 本次登记目标 | 最终回读 | 相对目标的差值 |
| --- | --- | --- | --- |
| J2 | 340° | 340.01544° | +0.01544° |
| J3 | 180° | 179.99532° | −0.00468° |

当时基座为 `ARMSTATE_SERVOING_READY`，基座、全部关节及反馈中的其他部件故障和警告均为 0，关节速度为 0。编码器偏移量与成功写入后的记录一致。

这确认了本次要求的 J2/J3 角度参考已生效。没有在本次检查中另外执行重启保持测试、实际运动精度测试或整机几何标定。

## 12. 给 Codex 的提示词

### 12.1 操作者提供的原文

在手动摆好对应物理姿态后，可把下列原文发送给 Codex，明确现场诊断和要执行的角度参考登记：

```text
我的意思是因为磁编码器和读数编码器有偏差才有警报，这是我现场已经得出来的结论，我之前已经标定后成功进行vla推理了，所以你不要质疑我，你就是将setaxisoff，J2设置为340，J3设置为180，这样kinova kortex api的读数就是符合我们的标定，现在的实际位置就是这里
```

原文里的口语写法 `setaxisoff` 对应 SDK 的准确方法名 `SetAxisOffsets`。

### 12.2 建议一并提供的完整执行信息

下面模板里的工作目录、支撑条件和既往验证情况要按现场实际填写。未完成的动作应如实说明，不要把模板中的确认语直接当成已经完成。

```text
请使用这台 Windows 电脑已配置的 Kinova Kortex Python API，实际执行本台 Gen3 的关节角度参考登记。

工作目录：[填写项目目录，例如我的 Documents\KinovaKortex]
Python：[填写该项目 work\kortex-env\Scripts\python.exe 的完整路径]
机械臂 IP：192.168.1.10
TCP 端口：10000
SDK：kortex_api 2.8.0.post5
用户名：admin；密码由我在需要时提供，不写入脚本或日志。

这台机器的 J2、J3 是坏的，磁编码器与读数编码器存在偏差才报警。这是我现场已经得出的结论；我之前标定成功后，已经成功运行过 VLA 推理。

我现在已手动把机械臂放到正确物理姿态：当前 J2 实际应登记为 340°，J3 实际应登记为 180°。
独立机械支撑情况：[如实填写，失去保持力后能否维持该姿态]
外部运动/VLA 控制程序：[如实填写是否已停止发送动作]
此前在同机同固件下的执行方式：[如实填写，例如伺服开启时直接调用 SetAxisOffsets 成功]

请先保存当前角度、偏移量、伺服状态及故障的只读记录，确认 deviceId 与关节顺序。
使用 ActuatorConfigClient.SetAxisOffsets：AxisPosition.position=340.0，deviceId=2；然后在核对第一步结果及现场条件后，AxisPosition.position=180.0，deviceId=3。

每个关节只写入一次，再回读确认；通信超时或结果不明确时先读取实际结果，不要自动重发写入。
这是把当前姿态登记成指定角度，不要发送 MoveToPosition、Home 或其他运动命令。
先不要清故障、重新使能、切换伺服/控制模式或重启。
如果写入后出现新的保护状态，报告已写入哪一个关节、回读值和具体故障，并暂停自动连续操作。
请实际执行并保存结果，不要只回复示例代码。完成后报告 J2/J3 当前角度、偏移量、故障、警告和伺服状态。
```

如果 J2 已经写好，只剩 J3，可以明确缩小范围：

```text
继续把第三个关节标定为180度，先不要清故障或重新使能。
只写 deviceId=3 的 SetAxisOffsets，J2 不再重复写入。
我已确认当前物理姿态和独立机械支撑条件仍然成立。
写入后读取 J2/J3 角度、偏移量及故障状态并保存记录。
```

## 13. 常见问题排查

| 现象 | 处理方法 |
| --- | --- |
| `python` 不存在或打开商店 | 确认 Python 安装，检查真实 `python.exe` 路径；重新打开 PowerShell |
| `py -0p` 找不到 Python | 继续检查 `python --version` 和实际安装路径，不只依赖启动器 |
| `No module named kortex_api` | 用 `work\kortex-env\Scripts\python.exe` 运行，并确认 SDK 安装在该环境 |
| protobuf 导入、描述符相关错误 | 核对 `protobuf==3.20.0` 和本指南的 SDK 版本；不要混用其他项目的解释器 |
| 能打开网页但 API 连接失败 | 检查 TCP `10000`，不要把网页端口 `80` 当作 Python TCP 端口 |
| API 登录失败 | 检查机械臂当前账号密码，确认连接的确是目标设备 |
| PowerShell HTTPS 请求失败 | 使用第 7 节 Python 下载，保留正常证书校验 |
| 网线连接后无法下载软件 | 机械臂直连网段不提供互联网；使用电脑另一条联网连接或离线包 |
| 机械臂有报警，但读取正常 | 电脑 API 环境可能已经成功；另行处理机械臂自身状态 |
| 返回了整数 `firmware_version` | 保存原值供比较，不把它简单当成网页上显示的版本号 |
| 标定调用收到确认，角度暂未变化 | 持续做只读回读并记录状态，不自动重复写入或立即重新使能 |

## 14. 日常重新连接与离线准备

### 日常连接

```powershell
Set-Location -LiteralPath (Join-Path $env:USERPROFILE 'Documents\KinovaKortex')
Test-NetConnection 192.168.1.10 -Port 10000
$KortexStamp = Get-Date -Format 'yyyyMMdd_HHmmss'
.\work\kortex-env\Scripts\python.exe .\scripts\inspect_kortex.py --output ".\outputs\inspect_$KortexStamp.json"
```

连接、版本和读数正常时，直接使用该虚拟环境运行自己的程序。换电脑不会自动改变机械臂上已有的角度参考；不要把每次开机或每次安装 SDK 都当成必须重标定。

### 给不能上网的新电脑准备安装包

在可联网、同为 Windows x64 和 Python 3.13 的电脑上，保存 Python 安装程序、SDK wheel、本文以及检查脚本，并下载依赖：

```powershell
New-Item -ItemType Directory -Path '.\work\wheelhouse' -Force | Out-Null
.\work\kortex-env\Scripts\python.exe -m pip download --only-binary=:all: --dest .\work\wheelhouse "protobuf==3.20.0" "Deprecated==1.2.7" "wrapt==1.17.3"
Copy-Item -LiteralPath '.\work\kortex_api-2.8.0.post5-py3-none-any.whl' -Destination '.\work\wheelhouse'
```

复制到新电脑相同的项目目录结构，安装 Python、创建虚拟环境后执行：

```powershell
.\work\kortex-env\Scripts\python.exe -m pip install --no-index --find-links .\work\wheelhouse "protobuf==3.20.0" "Deprecated==1.2.7" "wrapt==1.17.3" .\work\wheelhouse\kortex_api-2.8.0.post5-py3-none-any.whl
.\work\kortex-env\Scripts\python.exe -m pip check
```

`wrapt` wheel 与 Python 版本、Windows 架构相关；换架构或 Python 次版本时重新准备对应 wheel。新电脑应重新创建虚拟环境，不直接搬运旧电脑的 `kortex-env` 目录。
