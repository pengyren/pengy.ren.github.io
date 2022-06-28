---
title: Chat Services
description: NexT User Docs – Third-party Service Integration – Chat Services
---

### Chatra

[Chatra](https://chatra.com) is a live chat messenger app for your website.

{% tabs Chatra %}
<!-- tab Enable Chatra → -->
Visit [Dashboard](https://app.chatra.io/settings/general) to get your ChatraID.

```yml NexT config file
# Chatra Support
# See: https://chatra.com
# Dashboard: https://app.chatra.io/settings/general
chatra:
  enable: true
  async: true
  id: <ChatraID>
```

<!-- endtab -->

<!-- tab Activate sidebar button -->
After Chatra enabled, you can set `chat.enable` to `true` in {% label primary@NexT config file %}.

```yml NexT config file
# A button to open designated chat widget in sidebar.
# Firstly, you need enable the chat service you want to activate its sidebar button.
chat:
  enable: true
  icon: fa fa-comment
  text: Chat
```
<!-- endtab -->
{% endtabs %}

### Tidio

[Tidio](https://www.tidio.com/) is a live chat messenger app for your website.

{% tabs Tidio %}
<!-- tab Enable Tidio → -->
Visit [Dashboard](https://www.tidio.com/panel/dashboard) to get your Public Key.

```yml NexT config file
# Tidio Support
# See: https://www.tidio.com
# Dashboard: https://www.tidio.com/panel/dashboard
tidio:
  enable: true
  key: <Publick Key>
```

<!-- endtab -->

<!-- tab Activate sidebar button -->
After Tidio enabled, you can set `chat.enable` to `true` in {% label primary@NexT config file %}.

```yml NexT config file
# A button to open designated chat widget in sidebar.
# Firstly, you need enable the chat service you want to activate its sidebar button.
chat:
  enable: true
  icon: fa fa-comment
  text: Chat
```
<!-- endtab -->
{% endtabs %}

### Gitter

[Gitter](https://gitter.im) is a chat and networking platform that helps to manage, grow and connect communities through messaging, content and discovery.

{% tabs Gitter %}
<!-- tab Enable Gitter → -->
You need to create a community, then create a webapp room under that community.

```yml NexT config file
# Gitter Support
# For more information: https://gitter.im
gitter:
  enable: true
  room: <Community>/<Room Name>
```

<!-- endtab -->

<!-- tab Activate sidebar button -->
After Gitter enabled, you can set `chat.enable` to `true` in {% label primary@NexT config file %}.

```yml NexT config file
# A button to open designated chat widget in sidebar.
# Firstly, you need enable the chat service you want to activate its sidebar button.
chat:
  enable: true
  icon: fa fa-comment
  text: Chat
```
<!-- endtab -->
{% endtabs %}
