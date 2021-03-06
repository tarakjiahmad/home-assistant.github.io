---
layout: page
title: "Push Notifications"
description: "Instructions how to use the HTML5 push notifications platform from Home Assistant."
date: 2016-08-17 21:58
sidebar: true
comments: false
sharing: true
footer: true
logo: html5.png
ha_category: Notifications
ha_release: 0.27
---

The `html5` notification platform enables you to receive push notifications to Chrome or Firefox, no matter where you are in the world. `html5` also supports Chrome and Firefox on Android, which enables native-app-like integrations without actually needing a native app.

To enable this platform, add the following lines to your `configuration.yaml` file:

```yaml
# Example configuration.yaml entry
notify:
  - name: NOTIFIER_NAME
    platform: html5
    gcm_api_key: 'gcm-sender-key'
    gcm_sender_id: 'gcm-sender-id'
```

Configuration variables:

- **name** (*Optional*): Setting the optional parameter `name` allows multiple notifiers to be created. The default value is `notify`. The notifier will bind to the service `notify.NOTIFIER_NAME`.
- **gcm_api_key** (*Required if pushing to Chrome*): The API key provided to you by Google for Google Cloud Messaging (GCM). Required to push to Chrome.
- **gcm_sender_id** (*Required if pushing to Chrome*): The sender ID provided to you by Google for Google Cloud Messaging (GCM). Required to push to Chrome.

### {% linkable_title Getting ready for Chrome %}

- Create new project at [https://console.cloud.google.com/home/dashboard](https://console.cloud.google.com/home/dashboard).
- Go to [https://console.cloud.google.com/apis/credentials/domainverification](https://console.cloud.google.com/apis/credentials/domainverification) and verify your domain.
- After that, go to [https://console.firebase.google.com](https://console.firebase.google.com) and select import Google project, select the project you created.
- Then, click the clogwheel on top left and select "Project settings".
- Select Cloud messaging tab if under server key is button Regenerate key, click that.


### {% linkable_title Requirements %}

The `html5` platform can only function if all of the following requirements are met:

* You are using Chrome and/or Firefox on any desktop platform, ChromeOS, or Android.
* Your Home Assistant instance is exposed to the world.
* If using a proxy, HTTP basic authentication must be off for registering or unregistering for push notifications. It can be re-enabled afterwards.
* `pywebpush` must be installed. `libffi-dev`, `libpython-dev`, and `libssl-dev` must be installed prior to `pywebpush` (i.e. `pywebpush` probably won't automatically install).
* You have configured SSL for your Home Assistant. It doesn't need to be configured in Home Assistant though, i.e. you can be running [NGINX](/ecosystem/nginx/) in front of Home Assistant and this will still work. The certificate must be trustworthy (i.e. not self signed).
* You are willing to accept the notification permission in your browser.

### {% linkable_title Setting up %}

Assuming you have already added the platform to your configuration:

1. Open Home Assistant in Chrome or Firefox.
2. Assuming you have met all the [requirements](#requirements) above, you should see a new slider in the sidebar labeled Push Notifications.
3. Slide it to the on position.
4. Within a few seconds you should be prompted to allow notifications from Home Assistant.
5. Assuming you accept, that's all there is to it!
6. (Optional, but highly recommended!) Open the `html5_push_registrations.conf` file in your configuration directory. You will see a new entry for the browser you just added. Rename it from `unnamed device` to a name of your choice, which will make it easier to identify later. _Do not change anything else in this file!_ You need to restart Home Assistant after making any changes to the file.

### {% linkable_title Usage %}

The `html5` platform accepts a standard notify payload. However, there are also some special features built in which you can control in the payload.

Any JSON examples below can be [converted to YAML](https://www.json2yaml.com/) for automations.

#### {% linkable_title Actions %}

Chrome supports notification actions, which are configurable buttons that arrive with the notification and can cause actions on Home Assistant to happen when pressed. You can send [up to 2 actions](https://cs.chromium.org/chromium/src/third_party/WebKit/public/platform/modules/notifications/WebNotificationConstants.h?q=maxActions&sq=package:chromium&dr=CSs&l=14).

```json
{
  "message": "Anne has arrived home",
  "data": {
    "actions": [
      {
        "action": "open",
        "icon": "/static/icons/favicon-192x192.png",
        "title": "Open Home Assistant"
      },
      {
        "action": "open_door",
        "title": "Open door"
      }
    ]
  }
}
```

#### {% linkable_title Data %}

Any parameters that you pass in the notify payload that aren't valid for use in the HTML5 notification (`actions`, `badge`, `body`, `dir`, `icon`, `lang`, `renotify`, `requireInteraction`, `tag`, `timestamp`, `vibrate`) will be sent back to you in the [callback events](#automating-notification-events).

```json
{
  "title": "Front door",
  "message": "The front door is open",
  "data": {
    "my-custom-parameter": "front-door-open"
  }
}
```

#### {% linkable_title Tag %}

By default, every notification sent has a randomly generated UUID (v4) set as its _tag_ or unique identifier. The tag is unique to the notification, _not_ to a specific target. If you pass your own tag in the notify payload you can replace the notification by sending another notification with the same tag. You can provide a `tag` like so:

```json
{
  "title": "Front door",
  "message": "The front door is open",
  "data": {
    "tag": "front-door-notification"
  }
}
```

Example of adding a tag to your configuration. This won't create new notification if there already exists one with the same tag.

```yaml
  - alias: Push/update notification of sensor state with tag
    trigger:
      - platform: state
        entity_id: sensor.sensor
    action:
      service: notify.html5
      data_template:
        message: "Last known sensor state is {% raw %}{{ states('sensor.sensor') }}{% endraw %}."
      data:
        data:
          tag: 'notification-about-sensor'
```

#### {% linkable_title Targets %}

If you do not provide a `target` parameter in the notify payload a notification will be sent to all registered targets as listed in `html5_push_registrations.conf`. You can provide a `target` parameter like so:

```json
{
  "title": "Front door",
  "message": "The front door is open",
  "target": "unnamed device"
}
```

`target` can also be a string array of targets like so:

```json
{
  "title": "Front door",
  "message": "The front door is open",
  "target": ["unnamed device", "unnamed device 2"]
}
```

#### {% linkable_title Overrides %}

You can pass any of the parameters listed [here](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerRegistration/showNotification#Parameters) in the `data` dictionary. Please note, [Chrome specifies](https://cs.chromium.org/chromium/src/third_party/WebKit/public/platform/modules/notifications/WebNotificationConstants.h?q=maxActions&sq=package:chromium&dr=CSs&l=21) that the maximum size for an icon is 320px by 320px, the maximum `badge` size is 96px by 96px and the maximum icon size for an action button is 128px by 128px.

#### {% linkable_title URL %}

You can provide a URL to open when the notification is clicked by putting `url` in the data dictionary like so:

```json
{
  "title": "Front door",
  "message": "The front door is open",
  "data": {
    "url": "https://google.com"
  }
}
```

If no URL or actions are provided, interacting with a notification will open your Home Assistant in the browser. You can use relative URLs to refer to Home Assistant, i.e. `/map` would turn into `https://192.168.1.2:8123/map`.

### {% linkable_title Automating notification events %}

During the lifespan of a single push notification, Home Assistant will emit a few different events to the event bus which you can use to write automations against.

Common event payload parameters are:

| Parameter | Description                                                                                                                                                                                                                                           |
|-----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `action`  | The `action` key that you set when sending the notification of the action clicked. Only appears in the `clicked` event.                                                                                                                               |
| `data`    | The data dictionary you originally passed in the notify payload, minus any parameters that were added to the HTML5 notification (`actions`, `badge`, `body`, `dir`, `icon`, `lang`, `renotify`, `requireInteraction`, `tag`, `timestamp`, `vibrate`). |
| `tag`     | The unique identifier of the notification. Can be overridden when sending a notification to allow for replacing existing notifications.                                                                                                               |
| `target`  | The target that this notification callback describes.                                                                                                                                                                                                 |
| `type`    | The type of event callback received. Can be `received`, `clicked` or `closed`.                                                                                                                                                                        |

You can use the `target` parameter to write automations against a single `target`. For more granularity, use `action` and `target` together to write automations which will do specific things based on what target clicked an action.

#### {% linkable_title received event %}

You will receive an event named `html5_notification.received` when the notification is received on the device.

```yaml
- alias: HTML5 push notification received and displayed on device
  trigger:
    platform: event
    event_type: html5_notification.received
```

#### {% linkable_title clicked event %}

You will receive an event named `html5_notification.clicked` when the notification or a notification action button is clicked. The action button clicked is available as `action` in the `event_data`.

```yaml
- alias: HTML5 push notification clicked
  trigger:
    platform: event
    event_type: html5_notification.clicked
```

or

```yaml
- alias: HTML5 push notification action button clicked
  trigger:
    platform: event
    event_type: html5_notification.clicked
    event_data:
     action: open_door
```

#### {% linkable_title closed event %}

You will receive an event named `html5_notification.closed` when the notification is closed.

```yaml
- alias: HTML5 push notification clicked
  trigger:
    platform: event
    event_type: html5_notification.closed
```
