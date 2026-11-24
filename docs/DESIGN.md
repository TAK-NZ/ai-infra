# AI Infra design: a standalone TAK-GPT bot

> **Status: design, not implemented.** Nothing in this document is deployed
> from this repository yet. The existing TAK-GPT bots (RELAY, SENTINEL) still
> run as a plugin inside TAK Server, built from `tak-infra`. Check the issues in
> this repository for current status.
>
> Written 2026-11-24 after a spike on the Dev (Auckland) deployment. The
> [AgentCore migration plan](roadmap/AGENTCORE-MIGRATION.md) is a companion
> document and has been moved here from `tak-infra`.

## Summary

- The TAK-GPT chat bot stops being a Java plugin inside TAK Server. It becomes a
  separate service that connects to TAK Server as an ordinary client with its own
  certificate.
- This repository (`ai-infra`) holds the bot, the AI resources it uses (agent,
  knowledge bases, tool Lambdas), and the enrollment logic. The plugin and the
  plugin build are removed from `tak-infra`.
- The Bedrock resources run in Sydney (`ap-southeast-2`) for deployments in
  Auckland, because Auckland has no Bedrock Agents, Knowledge Bases or AgentCore.
  The bot runs next to TAK Server.

## Why leave the plugin

The plugin is a fork of `raytheonbbn/tak-gpt` (about 2,000 lines of our own Java
plus vendored libraries, with providers we do not use: Ollama, OpenAI, Vertex,
Google ADK). It is compiled into the TAK Server image.

- **Deploys.** A prompt, tool or model change means rebuilding the TAK Server
  image and restarting the ECS service, which drops every client.
- **Coupling.** The build needs the TAK Server plugin jars and artifacts.tak.gov
  credentials, and is tied to the TAK Server version.
- **Isolation.** A bot fault shares a JVM with TAK Server. Its LLM calls use a
  fixed 4-thread pool. The bot uses the TAK Server task role for Bedrock.
- **Fit with the migration.** The AgentCore agent is Python (Strands). The Java
  `BedrockChatManager` (592 lines, including the RETURN_CONTROL loop) would
  otherwise have to be rewritten against a new API anyway.
- **Access control.** The plugin can send to any group, including `__ANON__`. A
  service account only has the groups it was given.

## Scope of this repository

| In `ai-infra` | Stays in other repos |
|---|---|
| Bot service (TAK client, chat handling, presence, markers) | TAK Server, enrollment endpoint: `tak-infra` |
| Bot enrollment and certificate handling | Authentik: `auth-infra` |
| Agent / AgentCore runtime, Strands app | Group and channel management: `TAKTeamManager` |
| Knowledge bases (S3 Vectors), documents bucket | VPC, ECS cluster, ECR, KMS, S3: `base-infra` |
| Action Lambdas (GeoJSON, ArcGIS geocode, CloudTAK) | SitRep generator: `utils-infra` (calls `InvokeModel` directly) |
| System prompts, KB content helpers | |

The earlier decision to keep this work in `tak-infra` rested on the agent's
RETURN_CONTROL contract being coupled to the in-server plugin. With the plugin
gone, that coupling disappears and the agent, tools and bot belong together.

## Architecture

```
                 Auckland (ap-southeast-6)                          Sydney (ap-southeast-2)
 ┌──────────────────────────────────────────────┐        ┌────────────────────────────────┐
 │  TAK Server (tak-infra)                      │        │  AiInfraBedrock stack          │
 │     ▲  8089 CoT stream (mTLS)                │        │   - Knowledge bases (S3 Vec.)  │
 │     │  8446 enrollment (password, once)      │        │   - documents bucket + KMS     │
 │  ┌──┴────────────────────┐   InvokeAgent /   │        │   - AgentCore runtime (Strands)│
 │  │ AiInfra: bot service  │ ─ InvokeAgentRuntime ───►   │   - Action Lambdas, secrets    │
 │  │ (ECS Fargate, 1 task) │   (public AWS API)│        └────────────────────────────────┘
 │  └──────────┬────────────┘                   │
 │             │ secrets: bot password, cert    │        Demo (already in Sydney): one region,
 │  Authentik (auth-infra): service account     │        one stack, `bedrock.region` unset.
 └──────────────────────────────────────────────┘
```

- The Bedrock stack exists only when `bedrock.region` differs from the stack
  region. This is the layout decided in [#6](https://github.com/TAK-NZ/ai-infra/issues/6) (formerly tak-infra#182): one Bedrock
  stack per environment, its own documents bucket (a knowledge base needs its
  source bucket in its own region), Lambdas and agent. No shared stack.
- Cross-region CloudFormation references do not work. Values pass through stack
  outputs or SSM parameters.
- The bot is not placed in Sydney. It needs TAK Server, Authentik and the base
  layer, so a Sydney bot would need its own VPC, NAT, ECS cluster and ECR, and
  would reach TAK Server over the internet instead of the VPC. Only the parts
  that are region-constrained move.
- Residency: chat text reaches Sydney through the agent call. This was accepted
  in #6 (RAG documents, vectors and agent sessions may be
  stored in Sydney).
- Models: `au.anthropic.claude-sonnet-5-5` is available in Sydney. In Auckland
  it is only available as `global.`. This is a further reason for the agent to
  run in Sydney.

## Bot identity and enrollment

Each bot (RELAY, SENTINEL) has its own Authentik service account, its own
certificate and its own callsign. They do not share a connection.

### Enrollment (verified)

The bot enrolls with its service account password, the way CloudTAK creates
machine users:

1. `GET https://<tak>:8446/Marti/api/tls/config` returns `O` and `OU`
   (`TAK` and `TAK Unit` on Dev).
2. Generate a key and a CSR whose subject is `O=<O>, OU=<OU>, CN=<username>`.
   Without `O` and `OU` the server returns a 500.
3. `POST /Marti/api/tls/signClient/v2?clientUid=<username> (ETL)&version=3` with
   HTTP Basic auth (`username:password`), `Content-Type: text/plain`, and the PEM
   CSR as the body. The JSON response has `signedCert` (base64, no PEM
   header/footer) and `ca0` to `ca2`.
4. Connect to port 8089 with the cert and key. Server certificate verification
   was skipped in the spike; the bot must verify it against the CA chain.

The `version` parameter makes TAK Server add the channels OID to the cert (see
"Open items", group activation).

The issued cert was valid for 365 days plus 12 hours.

### Group membership

The bot sees and reaches only the clients it shares a group with. The sender's group set is
server-side information. A streaming client is assumed not to receive it (not
checked), so group membership is the access control: whoever can reach the bot is whoever shares a group with it.
The plugin could reply in the sender's groups regardless of its own membership.

The existing plugin config already attaches the bot to a group such as
"Coordination - All of New Zealand". How many users are in that group
decides who can reach the bot. This needs to be checked before cutover.

Directional groups follow the TAK `_READ` / `_WRITE` suffix convention. In
TAK Server's subscription data, `IN` means the client writes into the group and
`OUT` means it reads from it. So `tak_x_WRITE` appears as `x:IN` and `tak_x_READ`
as `x:OUT`.

### Who creates the account and group memberships (open decision)

| Option | For | Against |
|---|---|---|
| **A. TAKTeamManager** | Already provisions Authentik service accounts for channels (`provision_bch_service_account`, rotate password, delete), refuses to take over accounts it did not create, handles Authentik rate limits and has audit. Owns group membership, so no reconcile conflict. | Does not exist as IaC. A bot needs the model to fit (BCH/UTL channel service account, or a new type). How the password reaches this repo is unknown; may be shown once to an admin. |
| **B. Deploy-time provisioning from this repo (CDK custom resource or Lambda)**, driven by an S3 or CDK config of the groups the account must join | Declarative, per environment. | A second writer of Authentik group membership. TeamManager reconcilers may remove members they do not expect (their docs describe such an incident). Needs an Authentik write token, ideally restricted to adding members to listed groups. Not verified that Authentik can restrict a token that way. |

Rule for either option: the running bot never holds an Authentik write token. It
only holds its own password. The bot reads untrusted chat text, so it must not
be able to change groups.

## Spike results (Dev Auckland, 2026-11-24)

Run from a machine outside AWS Auckland, against the public NLB, with five
throwaway service accounts and six test groups. All of them were deleted
afterwards and the five certs were revoked (HTTP 200 each; the records still
show in the cert list, revoked state not re-checked).

| Question | Result |
|---|---|
| Enrollment with a password over 8446 | Works. |
| Cert validity | 365 days plus 12 hours. |
| Streaming over 8089 as a plain TLS client | Works. A small Python script is enough. |
| DM routing by `dest callsign` between connected clients | Works both ways. |
| All-chat | Works. |
| Presence | A client sees presence of clients it shares a group with. |
| Groups enforced | Yes. With no shared group, DMs and all-chat are not delivered. |
| `_WRITE`-only sender and `_READ`-only receiver | Write-only sender reaches the reader. The reader's DM and all-chat to the sender are dropped silently. The write-only client sees no presence. |

Other findings:

1. **New group memberships take up to about 4 minutes to reach TAK Server.** In
   one run a new user showed only `__ANON__` for roughly 4 minutes after being
   added to a group, then received the groups on its next login. The Authentik
   LDAP outpost already knew the user. A client only gets its groups at login.
   Whether an open session picks up a change was not tested.
2. **A client with no groups lands in `__ANON__`**, where every other
   group-less client also is. They could all chat. A bot whose groups did not
   load would be visible there, and would answer.
3. **Undeliverable DMs return a `b-t-f-s` event to the sender**, a copy of the
   message. It appeared when there was no shared group and not otherwise
   (observed; its meaning was not confirmed in the TAK Server source).
4. Chat text from other clients arrives XML-escaped (`->` as `-&gt;`).
5. TAK Server logs `is not a member of any group` as a WARN on each enrollment.
   This is harmless: enrollment does not need groups.
6. Each enrollment adds a row to TAK Server's cert list
   (`/Marti/api/certadmin/cert`).

## Bot behaviour to port from the plugin

From `plugins/tak-gpt` in `tak-infra`:

- Accept chat (`b-t-f`) whose destination callsign is the bot's.
- Send delivered and read receipts (`b-t-f-d`, `b-t-f-r`) immediately.
- Call the agent, with up to two attempts, and reply as a DM to the sender's
  device UID. Run LLM calls off the receive thread.
- Session id `<botName>:<senderUid>`. Sender UID comes from `<chatgrp uid0>`,
  or `remarks sourceID` if present (WinTAK). The callsign is display-only.
- Pass the sender's position to the agent when it is non-zero.
- Create map markers (RETURN_CONTROL `create_tak_map_marker`). The marker is
  addressed to the requester only. In the new service the bot sends the CoT
  itself.
- Broadcast presence every 8 seconds with the bot's configured position,
  group colour, role and CoT type.
- Per-bot settings: agent id or runtime id and alias, knowledge base ids and
  `knowledgeBaseResults`, position, group name, role, CoT type, region.

Not portable: the sender's group set (`msg.getGroupsList()`), which the plugin
passed to its tool handler. See the group section above.

## Open items

1. **Group activation.** The `tak-infra` env files for Test and Demo set `x509useGroupCache=true` and
   `x509useGroupCacheDefaultActive=true`, but not `DefaultUpdatesActive`
   (it is not plumbed through `createCoreConfig.sh`; Dev (Auckland) runs the Demo config). A group
   added to an account after its first login arrives inactive, and the bot would
   silently ignore it. Certs enrolled without the `version` parameter lack the
   channels OID and may skip the cache, which would keep all LDAP groups active.
   Not tested. Needs a test on Dev before the group-change process is fixed.
2. **Startup check.** After connecting, the bot should confirm it holds the
   expected groups (for example by asking TAK Server for its subscription or
   groups with its own cert; not tested) and reconnect until it does. It must
   not answer while in `__ANON__` only.
3. **Cert lifetime.** Enroll once, store the cert in Secrets Manager, renew at
   about 60 days remaining (the same pattern as `admin-cert-rotate.sh`). Do not
   enroll on every start, because each enrollment adds a cert record. A
   connection held past 365 days would break at expiry.
4. **Single instance.** Two sessions with the same UID would overlap during an
   ECS rolling deploy. Use stop-then-start (max 100%, min healthy 0%) or accept a
   short gap. Behaviour of TAK Server with two sessions on one UID was not
   tested.
5. **Monitoring.** The plugin could not be offline separately from TAK Server;
   the bot can. Needs a connection-state metric and alarm.
6. **Reachability policy.** Today anyone on the server can message the bot.
   After the change only members of its groups can, and only while they keep
   those groups active in ATAK. Restricted bots (SENTINEL) need a restricted group.
   Check the membership policy of "Coordination - All of New Zealand".
7. **Network path.** The TAK NLB is internet-facing on 8089 and 8446. Prefer a
   VPC-internal path for the bot.
8. **Abuse and cost.** Per-sender rate limits and LLM cost caps.
9. **Existing Bedrock resources.** The knowledge bases (S3 Vectors), tool
   Lambdas and the ArcGIS secret are in the `tak-infra` stack today. Removing
   the constructs from `tak-infra` would make CloudFormation delete them. They
   need to be retained and imported, or recreated and re-synced.
10. **AgentCore in `ap-southeast-2`.** The AgentCore endpoints resolve in
    `ap-southeast-2` and `us-west-2` and not in `ap-southeast-6` or
    `ap-southeast-4`. That is not proof the service works; verify as part of the
    AgentCore migration.

## Migration order

The bot change and the AgentCore change should not land together.

1. **Scaffold this repo** from the `media-infra` shape (stack config, CI
   workflows, tests).
2. **Provisioning decision** (option A or B above), and the open-item tests on
   Dev (group activation, startup check).
3. **Bot service** in this repo, calling the existing Classic agent
   (`bedrock-agent-runtime`). Run it under a different callsign next to the
   plugin bot in Dev.
4. **Cut over** per environment: disable the plugin bot, enable the new one.
5. **Remove the plugin from `tak-infra`**: `plugins/tak-gpt`, the plugin stage and
   `plugin-build.gradle` in the TAK Server Dockerfile, the plugin config upload
   and download scripts and their calls in `start-tak.sh`, the `plugins/tak-gpt`
   path in the content hash of both build workflows, the S3 `takserver-plugins/`
   config, the Bedrock permissions on the TAK Server task role, and
   `docs/TAK_GPT_PLUGIN.md`. `<plugins/>` in CoreConfig stays empty.
6. **Move the Bedrock resources** out of `tak-infra` with retain and import
   (open item 9), and add the Sydney stack for Auckland deployments.
7. **AgentCore migration** (Strands runtime, then retire Classic).
8. **CloudTAK-awareness tool.**

## Naming and layout (proposal)

- Repository: `ai-infra`.
- Stacks: `TAK-<env>-AiInfra` (bot, in the TAK region) and
  `TAK-<env>-AiInfraBedrock` (Sydney, only when `bedrock.region` differs).
- The user-facing name stays "TAK-GPT"; the bots keep their names and callsigns.
