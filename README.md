# AI Infrastructure

<p align='center'>AI features for TAK.NZ: chat bot, agents, knowledge bases and tools</p>

> **Status: design phase.** No code yet. The TAK-GPT bots (RELAY, SENTINEL) still
> run as a plugin inside TAK Server, deployed from
> [tak-infra](https://github.com/TAK-NZ/tak-infra). This repository will replace
> that plugin with a standalone bot service and take over the Bedrock resources.

## Overview

This repository will deploy the AI layer of a TAK.NZ stack:

- the TAK-GPT chat bot, a separate service that connects to TAK Server as a normal
  client with its own certificate
- the agent (Amazon Bedrock AgentCore, Strands SDK), knowledge bases (Amazon S3
  Vectors) and the tool Lambdas the agent calls
- enrollment of the bot against TAK Server and Authentik

It is targeted at the deployment of [TAK.NZ](https://tak.nz) via a CI/CD pipeline.

## Documents

- [Design](docs/DESIGN.md): why the bot leaves TAK Server, architecture, the
  verified enrollment flow, open items and migration order
- [AgentCore migration plan](docs/roadmap/AGENTCORE-MIGRATION.md): Bedrock Agents
  Classic to AgentCore (moved here from tak-infra)

## Layers

Each stack is deployed from its own repository. For the layer diagram and
deployment order see the [TAK.NZ organization overview](https://github.com/TAK-NZ).
This layer needs the base, auth and TAK infrastructure layers.

## License

AGPL-3.0, see [LICENSE](LICENSE).
