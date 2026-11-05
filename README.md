# Yeelight 奇光板接入 Apple 家庭：成功配置方法

中文 | [English](README.en.md)

这份记录介绍如何将 Yeelight 智能奇光板（`yeelink.light.plate2`）通过 Home Assistant 接入 Apple 家庭，并使用家庭 App **原生的颜色、色温和亮度控件**。

已在一台 LAN 固件版本为 `16` 的奇光板上通过 Mac 家庭 App 实测：选择绿色后灯板变绿；选择冷白色温后灯板变白；亮度从约 31% 调到 70% 后灯板变亮。

## 为什么需要适配

这台灯板的 Yeelight LAN 接口会接受并报告 HSV 色彩命令，但实测灯板没有实际变色；直接发送 RGB 命令则可以变色。HomeKit Bridge 会向灯光实体发送 HS 色相和饱和度，因此在 Home Assistant 中建立一个模板灯，将 HS 转成 RGB，再转发给原始 Yeelight 灯光实体。色温和亮度继续转发给原始实体。

**连接路径：** Apple 家庭 → HomeKit Bridge → 模板灯 → Yeelight 集成 → 奇光板。

以下是这次实机成功使用的 **Home Assistant 界面配置**。不同灯具型号、固件或 HA 版本的表现可能不同；不要仅凭灯具返回 `ok` 或实体状态变化判断灯板已变色。

## 1. 准备原始灯光实体

1. 在 Yeelight App 中开启该灯具的局域网控制。
2. 在 Home Assistant 添加 [Yeelight 集成](https://www.home-assistant.io/integrations/yeelight/)，确认能够看到奇光板灯光实体。
3. 前往「开发者工具 → 状态」，记下这盏灯的实体 ID。

下文用 `light.yeelight_plate2` 代表**原始 Yeelight 实体**。实际操作时，将所有 `light.yeelight_plate2` 替换成你自己设备的实体 ID；不要替换成新建的模板灯 ID。

## 2. 创建模板灯

打开「设置 → 设备与服务 → 辅助元素 → 创建辅助元素 → Template → 灯光」。名称可以填写 `奇光板适配`。在对应字段中填写：

**状态**

```jinja
{{ is_state('light.yeelight_plate2', 'on') }}
```

**亮度**

```jinja
{{ state_attr('light.yeelight_plate2', 'brightness') | default(0, true) }}
```

**HS 颜色**

```jinja
{{ state_attr('light.yeelight_plate2', 'hs_color') | default((0, 0), true) }}
```

**色温**（模板灯界面使用 mired）

```jinja
{{ state_attr('light.yeelight_plate2', 'color_temp') | default(370, true) }}
```

接下来分别在五个「动作」区块中点「添加动作」，选择「灯光：打开」或「灯光：关闭」。添加后，在该动作右上角菜单选择「YAML 编辑」，将其内容分别替换为下面的动作。每一块都要替换目标实体 ID。

### 开启时动作

```yaml
action: light.turn_on
target:
  entity_id: light.yeelight_plate2
```

### 关闭时动作

```yaml
action: light.turn_off
target:
  entity_id: light.yeelight_plate2
```

### 设置级别时动作

```yaml
action: light.turn_on
target:
  entity_id: light.yeelight_plate2
data:
  brightness: "{{ brightness }}"
```

### 设置 HS 颜色时的动作

```yaml
action: light.turn_on
target:
  entity_id: light.yeelight_plate2
data:
  rgb_color: >-
    {% set sat = (s | float) / 100 %}
    {% set hue = (h | float) / 60 %}
    {% set ns = namespace(rgb=[]) %}
    {% for n in [5, 3, 1] %}
      {% set k = (n + hue) % 6 %}
      {% set v = (1 - sat * ([0, [k, 4-k, 1] | min] | max)) * 255 %}
      {% set ns.rgb = ns.rgb + [v | round(0) | int] %}
    {% endfor %}
    {{ ns.rgb }}
  brightness: >-
    {{ brightness | default(state_attr('light.yeelight_plate2', 'brightness') | default(128, true), true) }}
```

### 设置色温时动作

```yaml
action: light.turn_on
target:
  entity_id: light.yeelight_plate2
data:
  color_temp_kelvin: "{{ color_temp_kelvin }}"
  brightness: >-
    {{ brightness | default(state_attr('light.yeelight_plate2', 'brightness') | default(128, true), true) }}
```

检查五个动作和四个模板的目标后点「提交」。成功后，Home Assistant 会创建一个新的 `light` 实体，例如 `light.qi_guang_ban_gua_pei`。实际实体 ID 以自己的界面显示为准。

这些动作来自实机成功使用的辅助元素；上面仅将设备专属实体 ID 改成了占位符。模板灯支持色彩与色温，实际色温选择范围取决于模板灯和设备的能力设置。

## 3. 桥接到 Apple 家庭

1. 打开「设置 → 设备与服务 → 集成 → HomeKit Bridge」，配置用于这盏灯的桥接器。
2. 选择 `Light` 域和包含模式；在实体选择中**只选择刚创建的模板灯**，不要同时包含原始 Yeelight 灯，以免出现两个灯具。
3. 保存配置；如果家庭 App 没有及时刷新，重新加载该 HomeKit Bridge 或重新打开家庭 App。
4. 在家庭 App 中为新出现的灯设置名称和房间。

如果更换了桥接的实体，Apple 家庭可能把它当成新配件，原先使用旧配件的场景或自动化需要重新指定。不要删除其他灯具或其他桥接器。

## 4. 用实际灯板验证

1. 先把亮度调到容易观察的水平。在家庭 App 的颜色选择器中从红色切到绿色，确认灯板**实际变色**。
2. 打开色温页切换暖白和冷白，确认灯板实际变化。
3. 将亮度设置为两个差别明显的数值，确认实际变亮或变暗。
4. 检查开关和状态同步；修改配置或重启后可再次验证。

本次已由现场用户确认颜色、色温和亮度变化；没有对所有色点、其他型号、其他固件和重启后的所有场景做完整测试。

## LAN 不可用时

如果 Yeelight 集成报告无法读取初始状态，或者 LAN 返回 `client quota exceeded`，先检查是否存在频繁调节灯具的自动化，尝试给灯具断电重启，再重新加载 Yeelight 集成。此次通过已有的小米 MIoT 集成发送 `miIO.reboot` 也恢复了连接；该方法是否可用取决于设备和集成。该错误的具体原因不能只凭错误文本确定。

参考：[Home Assistant Template 灯光](https://www.home-assistant.io/integrations/template/#light) · [HomeKit Bridge](https://www.home-assistant.io/integrations/homekit/) · [Yeelight LAN 协议](https://www.yeelight.com/download/Yeelight_Inter-Operation_Spec.pdf)
