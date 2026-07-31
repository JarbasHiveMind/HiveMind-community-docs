# HiveMind Web Chat Interface and Bridges

HiveMind Bridges and the HiveMind Web Chat Interface let you connect AI assistants to other platforms and communicate with them through familiar channels.

## HiveMind Bridges

HiveMind Bridges connect external platforms to the HiveMind network. Each bridge acts as a terminal, so it can communicate with the HiveMind infrastructure. Bridges support several protocols, including Matrix, [Mattermost Bridge](https://github.com/OpenJarbas/HiveMind_mattermost_bridge), [HackChat Bridge](https://github.com/OpenJarbas/HiveMind-HackChatBridge), [DeltaChat Bridge](https://github.com/JarbasHiveMind/HiveMind-deltachat-bridge), and email, extending an assistant's reach to the channels users already use.

Each bridge acts as a secure intermediary, protecting the privacy of communications. A bridge keeps its own session and permissions, so it can answer specific users or follow custom rules. This lets you integrate an AI assistant into an existing communication platform.

## HiveMind Web Chat Interface

The [HiveMind Webchat](https://github.com/OpenJarbas/HiveMind-webchat) interface, built on [HiveMindJs](https://github.com/JarbasHiveMind/HiveMind-js), connects to the HiveMind network from a browser. This JavaScript library communicates directly with the HiveMind infrastructure when access keys are available in the browser environment. For example, a login page with HiveMind access keys can use HiveMindJs to establish a connection and give users access to the assistant.

If exposing HiveMind login keys in the browser is not acceptable for your security requirements, use a HiveMind Bridge instead. The bridge node connects to the [HiveMind network on a server](https://github.com/JarbasHiveMind/HiveMind-flask-template), while the browser talks only to the bridge. This keeps sensitive information off the browser.

## Integrating a chatbot with an existing platform

Consider an online platform where users already engage with your product. By hosting HiveMind-Core, ovos-core, and a HiveMind Bridge, you can integrate a chatbot into that platform.

The HiveMind Bridge acts as the intermediary between your platform and the HiveMind network. Users interact with the chatbot, ask questions, or request actions directly from your platform. The chatbot, backed by the HiveMind infrastructure, can respond, make recommendations, and support users.

Adding a chatbot this way streamlines customer support and automates parts of the user experience, while relying on HiveMind's existing security and permission model.

---
[Home](index.md)
