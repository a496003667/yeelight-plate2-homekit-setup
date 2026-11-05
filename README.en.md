# Yeelight Magi Light in Apple Home: Working Setup

[中文](README.md) | English

This guide connects a Yeelight Magi Light panel (`yeelink.light.plate2`) to Apple Home through Home Assistant, using the Home app's **native color, color temperature, and brightness controls**.

The setup was tested on one panel with LAN firmware version `16`, using the Home app on a Mac. The person next to the panel confirmed that it turned green when green was selected, changed to white when a cool white color temperature was selected, and became brighter when brightness was raised from approximately 31% to 70%.

## Why an adapter is needed

On the tested panel, the Yeelight LAN interface accepted HSV color commands and reported the new state, but the panel did not physically change color. Direct RGB commands did change the panel. HomeKit Bridge sends hue and saturation (HS) to a light entity, so this setup adds a template light in Home Assistant to convert HS to RGB and forward the result to the original Yeelight light. Color temperature and brightness are forwarded to the original light as well.

**Control path:** Apple Home → HomeKit Bridge → template light → Yeelight integration → panel.

The instructions below describe the **Home Assistant UI configuration** that worked on the tested device. Behavior may differ with other light models, firmware, or Home Assistant versions. An `ok` response or a changed entity state alone does not prove that the panel changed color.

## 1. Find the original light entity

1. Enable LAN control for the light in the Yeelight app.
2. Add the [Yeelight integration](https://www.home-assistant.io/integrations/yeelight/) in Home Assistant and confirm that the panel appears as a light entity.
3. Go to **Developer Tools → States** and note its entity ID.

In this guide, `light.yeelight_plate2` is a placeholder for the **original Yeelight entity**. Replace every occurrence of `light.yeelight_plate2` with your own device's entity ID. Do not use the ID of the new template light as its own target.

## 2. Create a template light

Open **Settings → Devices & services → Helpers → Create helper → Template → Light**. You can name it `Magi Light Adapter`. Enter the following in the corresponding fields:

**State**

```jinja
{{ is_state('light.yeelight_plate2', 'on') }}
```

**Brightness**

```jinja
{{ state_attr('light.yeelight_plate2', 'brightness') | default(0, true) }}
```

**HS color**

```jinja
{{ state_attr('light.yeelight_plate2', 'hs_color') | default((0, 0), true) }}
```

**Color temperature** (the template light field uses mireds)

```jinja
{{ state_attr('light.yeelight_plate2', 'color_temp') | default(370, true) }}
```

Next, in each of the five action sections, select **Add action**, then choose **Light: Turn on** or **Light: Turn off** as appropriate. Open the action's menu and select **Edit in YAML**, then replace its contents with the corresponding action below. Replace the target entity ID in each action.

### Actions on turn on

```yaml
action: light.turn_on
target:
  entity_id: light.yeelight_plate2
```

### Actions on turn off

```yaml
action: light.turn_off
target:
  entity_id: light.yeelight_plate2
```

### Actions on set level

```yaml
action: light.turn_on
target:
  entity_id: light.yeelight_plate2
data:
  brightness: "{{ brightness }}"
```

### Actions on set HS color

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

### Actions on set color temperature

```yaml
action: light.turn_on
target:
  entity_id: light.yeelight_plate2
data:
  color_temp_kelvin: "{{ color_temp_kelvin }}"
  brightness: >-
    {{ brightness | default(state_attr('light.yeelight_plate2', 'brightness') | default(128, true), true) }}
```

Check the targets in all five actions and four templates, then select **Submit**. Home Assistant will create a new `light` entity, such as `light.magi_light_adapter`; the actual ID is shown in your interface.

These actions come from the helper used in the successful physical test; only the device-specific entity ID has been replaced with a placeholder. The template light supports color and color temperature. The available temperature range depends on the capabilities configured for the template light and device.

## 3. Bridge the light to Apple Home

1. Open **Settings → Devices & services → Integrations → HomeKit Bridge** and configure the bridge used for this light.
2. Select the `Light` domain and include mode. In the entity selection, **select only the new template light**, not the original Yeelight light as well, to avoid duplicate accessories.
3. Save the configuration. If the Home app does not update promptly, reload that HomeKit Bridge or reopen the Home app.
4. Set the new light's name and room in the Home app.

Apple Home may treat a changed bridged entity as a new accessory. Scenes or automations that used the old accessory may need to be updated. Do not delete other lights or bridges.

## 4. Verify the physical panel

1. Set the brightness high enough to see clearly. In the Home app's color picker, switch from red to green and confirm that the **panel itself changes color**.
2. Switch between warm and cool white on the color temperature tab and confirm that the panel changes.
3. Choose two noticeably different brightness levels and confirm that the panel becomes brighter or dimmer.
4. Check on/off control and state synchronization; repeat the checks after configuration changes or a restart.

For the documented setup, the person on site confirmed changes in color, color temperature, and brightness. Not every color, other model, other firmware version, or post-restart scenario was tested.

## If LAN control is unavailable

If the Yeelight integration cannot read the initial state, or the LAN interface returns `client quota exceeded`, check for automations that change the light frequently. Try power cycling the light and reloading the Yeelight integration. In this case, sending `miIO.reboot` through an existing Xiaomi MIoT integration also restored the connection; availability depends on the device and integration. The error text alone does not establish its precise cause.

References: [Home Assistant Template light](https://www.home-assistant.io/integrations/template/#light) · [HomeKit Bridge](https://www.home-assistant.io/integrations/homekit/) · [Yeelight LAN protocol](https://www.yeelight.com/download/Yeelight_Inter-Operation_Spec.pdf)
