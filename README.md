# RFC: Internal JWT Minting for Agent Auto-Resume

Status: Draft

## Summary

**Agent auto-resume** is the mechanism that wakes a paused agent sandbox when a user attempts to connect to it. This RFC proposes how this will work natively for **interactive CLI and SSH sessions**: when a user executes a command, the system reads their authentication token, checks the Sandbox Custom Resource for the `agent-id` label, and explicitly resumes the agent on their behalf. 

Auto-resuming via **Web UI sessions** presents a different challenge. Browsers rely on session cookies. When a request comes in, the edge auth service validates and strips the cookie before routing traffic to the internal `sandbox-activator`. 

When the `sandbox-activator` receives traffic for a paused agent, it must forward a Resume request to the Control Plane (CP). The CP needs to authenticate this request. However, because the cookie was already stripped at the edge, the activator has no identity context to send to the CP. Without this, the CP cannot authenticate *who* is attempting to wake the agent, and because the CP only authenticates API Keys and JWTs (enforced via `pkg/middleware`), it rejects the request.

This document explores approaches to solve this problem. We must find a secure way to pass the user's identity to the Control Plane during Web UI sessions. Identifying the user is a strict requirement to ensure accurate billing, security, and clear audit traces.

The target access patterns are:
- **CLI/SSH.** A developer runs commands or SSHs directly into the agent. 
- **Web UI.** A user interacts with the OpenClaw or OpenCode sandbox proxy via a browser session.


## Flow 1: API-key auto-resume

When a user interacts with a sandbox by providing an external API key (e.g., for SSH, Exec, or File Ops), the system will use the following approach to auto-resume the agent.


**Flow Details:**
1. **Client Request:** A client (e.g., MCP router or SDK) sends a request to an **agent-backed sandbox** using an API key. This request hits the **sandbox-ingress-gateway** (an Envoy chain + forward proxy) in the Data Plane.
2. **Authorize:** The gateway makes an `ext_authz` call to the **auth-service** for authorization and liveness checks (operating on a tight 250ms budget).
3. **Upstream Routing:** If the agent is paused, the `auth-service` instructs the gateway to dynamically set the upstream destination to the **sandbox-activator**.
4. **Forwarding:** The gateway forwards the original request to the `sandbox-activator`, which uses hold, single-flight, and forward logic.
5. **CR Lookup:** The `sandbox-activator` queries the Kubernetes **Sandbox CR** (whose state is persisted in `etcd`) to check for the presence of the `agent-id` label.
6. **CR Response:** The label status is returned to the activator.
7. **Triggering CP:** The activator sends a resume request (passing along the caller's API key) through the **Istio internal gateway** (the DP-to-CP path secured with an internal CA) to the **aiagent-service** in the Control Plane. 
   - **If `agent-id` is present:** It calls the `ResumeAgent` API.
   - **If `agent-id` is missing:** It calls the standard `ResumeSandbox` API (returns **409 Conflict** if the sandbox is actually agent-backed).
   The CP service evaluates API key scopes, validates funds, and processes snapshot and restore annotations to unpause the agent.
8. **Holding & Proxying:** The activator holds the original request until the agent transitions to `Ready`. Once ready, it proxies the connection back to the **sandbox-ingress-gateway**.
9. **Re-Authorization:** The gateway receives the proxied request and once again makes an `ext_authz` call to the **auth-service**. 
10. **Final Routing:** Because the agent is now running, the auth-service allows the request and routes the traffic to `sandboxd` (port 44772) on the resumed agent.
11. **Response:** The response is finally returned to the client.

```mermaid
flowchart LR
    cl["client<br/>MCP router / SDK (api-key)"]:::store

    subgraph DP [Data Plane]
        cr[("sandbox CR<br/>(k8s etcd)")]:::store
        gw["sandbox-ingress-gateway<br/>Envoy chain + forward proxy"]:::dp
        az["auth-service ext_authz<br/>authz + liveness (250ms budget)"]:::dp
        act["sandbox-activator<br/>hold · single-flight · forward"]:::star
        igw["Istio internal gateway<br/>TLS 443 · internal CA"]:::dp
        sd["sandboxd :44772"]:::dp
    end

    subgraph CP [Control Plane]
        cp["aiagent-service<br/>ResumeAgent OR ResumeSandbox<br/>api-key scope · funds · snapshot + restore annotations"]:::cp
    end

    cl --->|"1 request to sb-abc"| gw
    gw --->|"2 authorize"| az
    az -..->|"3 suspended → instruct:<br/>upstream = activator"| gw
    gw --->|"4 forward original request"| act
    act --->|"5 check agent-id label"| cr
    cr -..->|"6 returns label status"| act
    act --->|"7 resume (caller's api-key)<br/>ResumeAgent if label exists<br/>ResumeSandbox if missing"| igw
    igw ---> cp
    act --->|"8 hold until Ready, then proxy"| gw
    gw --->|"9 authorize (again)"| az
    az -..->|"10 running (OK)"| gw
    gw --->|"11 route to sandbox"| sd
    sd -..->|"12 response"| gw
    gw -..->|"13 response"| cl

    classDef cp fill:#e7e6fb,stroke:#6b6be0,color:#20233a
    classDef dp fill:#cdeee7,stroke:#12a594,color:#10302b
    classDef star fill:#ffe6a7,stroke:#d99a1c,stroke-width:2px,color:#3a2c07
    classDef store fill:#e6e9ef,stroke:#5b6472,color:#20233a
```

## Flow 2: Web UI auto-resume

When resuming an **agent-backed sandbox** via a Web UI, the browser uses a session Cookie. The challenge here is that the Auth Service validates and strips this Cookie before forwarding the request to the `sandbox-activator`. 

Because the cookie is deleted at the edge, the `sandbox-activator` receives the request without any user identity. If the activator attempts to send a resume request to the Control Plane without these authentication fields, the request will be rejected.

To resolve this identity propagation issue, the following architecture defines how to securely pass the user's identity to the Control Plane.

### Auth Service Cookie JWT Extraction

The `auth-service` acts as a token-extraction layer, retrieving the existing JWT from the user's session cookie and forwarding it so the Control Plane can natively validate and authenticate the user.

**How it works:**
1. The **sandbox-ingress-gateway** receives the Web UI request and triggers `ext_authz` on the **auth-service**.
2. The `auth-service` validates the session cookie and **extracts the existing JWT directly from the cookie**. 
3. The `auth-service` injects this existing JWT into a custom header named `X-Neev-Resume-Token`. It explicitly avoids using the standard `Authorization` header so it doesn't accidentally overwrite or erase passwords needed by web apps running inside the user's sandbox (like Code Server). The `auth-service` also leaves the user's original session cookie attached to the request.
4. The gateway routes the suspended request to the `sandbox-activator`.
5. The `sandbox-activator` extracts the JWT from the `X-Neev-Resume-Token` header and securely forwards it to the **aiagent-service** (Control Plane) to authenticate the Resume request and wake the agent.
6. The activator holds the user's original HTTP connection open while it waits for the agent Pod to become `Ready`.
7. Once `Ready`, the activator proxies the connection (with the original session cookie still attached) back to the **sandbox-ingress-gateway**.
8. The gateway runs `ext_authz` again. Because the cookie was preserved, this functions as an ordinary cookie validation. The **auth-service** validates the request and issues an instruction to the gateway to remove the session cookie and any `X-Neev-Resume-Token` headers. The gateway then executes this instruction, physically stripping the headers before routing the final traffic to the untrusted `sandboxd` container.

```mermaid
flowchart LR
    cl["browser<br/>Web UI Session (cookie)"]:::store

    subgraph DP [Data Plane]
        gw["sandbox-ingress-gateway<br/>Envoy chain + forward proxy"]:::dp
        az["auth-service ext_authz<br/>validate cookie + extract JWT"]:::dp
        act["sandbox-activator<br/>hold · single-flight · proxy"]:::star
        igw["Istio internal gateway<br/>TLS 443 · internal CA"]:::dp
        sd["sandboxd :44772"]:::dp
    end

    subgraph CP [Control Plane]
        cp["aiagent-service ResumeAgent<br/>validates existing cookie JWT"]:::cp
    end

    cl --->|"1 GET / (cookie)"| gw
    gw --->|"2 authorize"| az
    az -..->|"3 suspended → instruct:<br/>keep cookie, inject extracted token,<br/>upstream = activator"| gw
    gw --->|"4 forward req (cookie + extracted token)"| act
    act --->|"5 resume (pass token, drop it)"| igw
    igw ---> cp
    cp -..->|"6 Agent awakened"| igw
    igw -..->|"7 Return success"| act
    act --->|"8 hold until Ready, proxy (cookie)"| gw
    gw --->|"9 authorize (cookie pass)"| az
    az -..->|"10 OK → instruct:<br/>strip cookie & resume token"| gw
    gw --->|"11 strip headers & route to sandbox"| sd
    sd -..->|"12 response"| gw
    gw -..->|"13 response"| cl

    classDef cp fill:#e7e6fb,stroke:#6b6be0,color:#20233a
    classDef dp fill:#cdeee7,stroke:#12a594,color:#10302b
    classDef star fill:#ffe6a7,stroke:#d99a1c,stroke-width:2px,color:#3a2c07
    classDef store fill:#e6e9ef,stroke:#5b6472,color:#20233a
```

## Architectural & Security Implications

Because this approach reuses the Data Plane's `uisession` token for Control Plane authentication, the following architectural changes and trade-offs must be accepted:

1. **Key Distribution to Control Plane:** Currently, the Control Plane (`aiagent-service` and `tenant-service`) only possesses the `CONNECT_TOKEN_KEY`. To verify the extracted cookie token, we must securely inject the Data Plane's `UI_SESSION_KEY` into the CP's configuration.
2. **Middleware Updates:** The Control Plane normally authenticates internal requests using standard API Keys or CP-specific tokens. The authentication middleware must be modified to explicitly accept, parse, and trust `uisession` claims so the CP can authenticate the user and check billing.
3. **Tenant Service Blast Radius:** The extracted token is an 8-hour master session key. While it is strictly scoped to one specific sandbox (so attackers cannot access other sandboxes), it lacks a strict `scope=resume` restriction. If intercepted on the internal network, an attacker could bypass the `tenant-service`'s API protections and execute unauthorized lifecycle operations (e.g., permanently deleting or pausing) on that specific sandbox directly from the CP.

## Security
- The extracted JWT is a long-lived master session token. Care must be taken to ensure it is not logged or leaked internally.
- The `X-Neev-Resume-Token` header and the raw session cookie are strictly stripped by the gateway before requests are finally routed into the untrusted `sandboxd` environment.

## Rejected Alternatives

| Approach | Concept | Reason for Rejection |
| :--- | :--- | :--- |
| **Auth Service Header Injection** | `auth-service` validates and strips cookie, injecting an `X-Neev-User-ID: <uuid>` plain-text header. Activator forwards this User ID header to the CP. | **CP Validation Failure:** The Control Plane authenticates requests using API Keys or JWTs. It will reject the request because it cannot authorize actions based purely on a plain-text User ID. |
| **Server API Key Bypass** | Activator bypasses user validation by injecting a master `Server API Key` to authenticate the Resume request against the CP. | **Breaks Tracing & Billing:** While this bypasses CP validation, it destroys user-level tracing and accurate billing attribution. The Control Plane logs the action under the Server identity, completely losing the identity of the user who initiated the resume. |
| **Authorization Header Overwrite** | `auth-service` injects the internal JWT into the `Authorization` header instead of a custom header. | **Breaks Sandbox Apps:** Web applications running inside the user's sandbox (e.g., Code Server) may rely on their own `Authorization` headers. Overwriting or stripping this header at the edge breaks native application functionality. |
