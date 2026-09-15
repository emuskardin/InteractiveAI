# InteractiveAI Assistant Platform
**An interactive AI Assistant Platform for Real Time operations**

_Frontend_ 
​ [![Node](https://img.shields.io/badge/Node-339933?style=plastic&logo=nodedotjs&logoColor=fff)](https://nodejs.org) [![Vue](https://img.shields.io/badge/Vue-35495E?style=plastic&logo=vuedotjs&logoColor=fff)](https://vuejs.org) [![Vite](https://img.shields.io/badge/Vite-%23646CFF.svg?style=plastic&logo=vite&logoColor=fff)](https://vitejs.dev) [![TypeScript](https://img.shields.io/badge/Typescript-%23007ACC.svg?style=plastic&logo=typescript&logoColor=fff)](https://www.typescriptlang.org) [![Leaflet](https://img.shields.io/badge/Leaflet-199900?style=plastic&logo=Leaflet&logoColor=fff)](https://leafletjs.com) [![Axios](https://img.shields.io/badge/Axios-671ddf?&style=plastic&logo=axios&logoColor=fff)](https://axios-http.com)

_Backend_ 
![Python](https://img.shields.io/badge/python-3670A0?style=plastic&logo=python&logoColor=ffdd54)
![Flask](https://img.shields.io/badge/flask-%23000.svg?style=plastic&logo=flask&logoColor=white)
![Postgres](https://img.shields.io/badge/postgres-%23316192.svg?style=plastic&logo=postgresql&logoColor=white)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=plastic&logo=docker&logoColor=white)
![Postman](https://img.shields.io/badge/Postman-FF6C37?style=plastic&logo=postman&logoColor=white)

<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="#prerequisites">Prerequisites</a></li>
        <li><a href="#install">Install</a></li>
        <li><a href="#configuration">Configuration</a></li>
        <li><a href="#manual-setup">Manual setup</a></li>
        <li><a href="#the-powergrid-expert-agent-api">The PowerGrid expert agent API</a></li>
      </ul>
    </li>
    <li><a href="#development">Development</a></li>
    <li><a href="#docs">Docs</a></li>

  </ol>
</details>

<!-- ABOUT THE PROJECT -->
## About The Project

InteractiveAI platform provides support in augmented decision-making for complex steering systems.
It is a prototype of a bi-directional virtual assistant, open in terms of industrial applications, in which it will be possible to evaluate the forms of exchange between the expert and an AI that learns continuously, both from the information flows received and the decisions made by the human. The platform will help and assist the operator of a complex operation to resolve incidents/faults in his industrial environment.

As it is designed, the platform is generic, it can be used for different use cases. As an example, the use case of managing **power line** overloads at **PowerGrid** (Réseau de Transport d'Electricité français) is provided. To install and run the PowerGrid simulator, please refer to the detailed guide available in the file PowerGrid simulator's [README](/usecases_examples/PowerGrid/README.md). This guide provides specific instructions for setting up and running the PowerGrid use case.

The platform uses the project **OperatorFabric** for notification management.


<!-- GETTING STARTED -->
## Getting Started

### Prerequisites

- Git, Docker Engine 27+, Docker Compose V2, `curl`, `python3`

### Install

```sh
git clone [repo-url] && cd InteractiveAI
cp config/dev/cab-standalone/.secrets.example config/dev/cab-standalone/.secrets
# edit .secrets — at minimum RL_AGENT_API_URL / RL_AGENT_API_TOKEN, see "Configuration"
./local_setup.sh
```

To use the local [A3S](a3s-service/README.md) service instead of a remote RL agent,
start it first and add `--a3s` — no `.secrets` change needed:

```sh
cd a3s-service && ./docker/local_setup.sh && cd ..   # see a3s-service/README.md
./local_setup.sh --a3s
```

`local_setup.sh` starts the backend, configures Keycloak, loads the OperatorFabric
resources, rebuilds the frontend and recommendation service from this source tree,
builds and starts the PowerGrid simulator, and verifies the recommendation path end
to end. It prints the URLs and credentials when it is done.

| Flag | Effect |
| --- | --- |
| *(none)* | full setup; asks what to do if containers from a previous run are up |
| `--clean` | tear those containers down first, no prompt |
| `--wipe` | tear down containers **and** volumes, no prompt |
| `--a3s [URL]` | take recommendations from an already-running [A3S](a3s-service/README.md); default URL `http://host.docker.internal:5010/api/v1/recommendation` |

It never starts A3S — start that yourself first, or `--a3s` aborts before touching
any container.

Then log in at http://localhost:3200 as `powergrid_user` / `test`, and in the
simulator (http://localhost:5122) pick server `http://host.docker.internal:3200/`.
Stop everything with `./local_stop.sh` (`--wipe` to drop the data volumes too).

The last step prints a warning for anything it could not verify — a stale nginx
upstream, an unreachable agent, a simulator/A3S payload mismatch. The UI still comes
up; the PowerGrid recommendation panel is what stays empty.

### Configuration

Everything lives in `config/dev/cab-standalone/.secrets` (gitignored,
`docker-compose.sh` sources it). The values that matter:

- `RL_AGENT_API_URL` / `RL_AGENT_API_TOKEN` — the agent producing PowerGrid
  recommendations. Options:
  - the **deep expert agent** on this host: `http://host.docker.internal:5123/api/v1/recommendation`
    (see [below](#the-powergrid-expert-agent-api)); it requires a token
  - the hosted one: `https://interactiveagent.passerelle.irt-systemx.fr/api/v1/recommendation`,
    also with a token
  - a local **[A3S](a3s-service/README.md)** — no token, and no need to set the URL:
    `./local_setup.sh --a3s` overrides it for that run
- `POWERGRID_SIMU_UPSTREAM` — where nginx forwards `/powergrid-simu/`. Local dev:
  `http://host.docker.internal:5122/`.
- `VITE_POWERGRID_SIMU` — the frontend's simulator endpoint; keep the same-origin
  proxy value `/powergrid-simu` (avoids CORS), or `false` to hide the PowerGrid UI.
  `VITE_RAILWAY_SIMU` / `VITE_ATM_SIMU` are the equivalents for the other use cases.
- `COGNITIVE_TOKEN` — bearer token for the INESCTEC cognitive API. nginx attaches it
  to every `/cognitive-api/` request, so it never reaches the browser. Empty is fine
  locally; you just lose that panel.

`host.docker.internal` is how the containers reach services on the host (simulator,
agent). Those services must listen on `0.0.0.0`, not only `127.0.0.1`.

If you hit CORS errors (the platform running without HTTPS), start a Chromium
browser with `--disable-web-security --user-data-dir="[some directory]"`.

Anything else: [troubleshooting guide](docs/troubleshooting.md).

### Manual setup

The same steps by hand, in order. `local_setup.sh` does all of them for you — use
this only when you need to run one in isolation.

1. **Backend** — `cd config/dev/cab-standalone && ./docker-compose.sh`
   (it writes `.env` from `.secrets` and brings the compose project up).
2. **Rebuild from source** — `docker compose up -d --build --force-recreate frontend
   cabrecommendation`, in the same directory. Step 1 reuses existing images, so
   without this your local changes are not in the running containers.
3. **Keycloak** — in the admin console (http://localhost:89/auth/admin,
   `admin`/`admin`), realm `dev`: set **Realm Settings → Frontend URL** to
   `http://localhost:3200/`, and add `http://localhost:3200/*` to
   **Clients → opfab-client → Valid Redirect URIs**.
4. **Restart the frontend** — `docker restart frontend`, so it picks up that change.
   Required before the next step.
5. **Resources** — `cd resources && ./loadTestConf.sh` (registers the use cases).
6. **Simulator** — `cd usecases_examples/PowerGrid && docker compose -f
   docker-compose.local.yml up -d --build app`. Use the `.local` compose file: the
   default one is the server config and binds the wrong port. See its
   [README](/usecases_examples/PowerGrid/README.md).
7. **Reload the gateway** — `docker exec frontend nginx -c /personal-conf/nginx.conf
   -s reload`. nginx caches upstream IPs at load, so containers recreated after it
   started answer 502 until this is done.

Check it: `curl localhost:3200/cab_recommendation/api/v1/health` should answer 200,
and the recommendation service must be able to reach `RL_AGENT_API_URL` from inside
its own container.

> **_NOTE:_** `cab` appears all over the project — it was the original name of
> InteractiveAI.

### The PowerGrid expert agent API

`cab_recommendation` calls the deep expert agent at `RL_AGENT_API_URL`, so that agent
must be running for recommendations to appear.

```sh
git clone https://github.com/ainetus/T2.1_deep_expert.git
cd T2.1_deep_expert
git checkout feat/api-auth-compose
```

Start it per that repo's README (that branch ships a Docker Compose and token auth),
on port **5123** and bound to `0.0.0.0`. Then set in `.secrets`:

```sh
export RL_AGENT_API_URL=http://host.docker.internal:5123/api/v1/recommendation
export RL_AGENT_API_TOKEN=<token configured in the expert agent>
```

The local alternative is [A3S](a3s-service/README.md), which serves the same API and
can project KPIs several timesteps ahead: start it from `a3s-service/` with
`./docker/local_setup.sh`, then run `./local_setup.sh --a3s` here.

### Default ports

This project is based on a microservice architecture. Every service run on a specific port. Some of the default ports are as fellow:
* Frontend: 3200
* Context Service: 5100
* Event Service: 5000
* Historic Service: 5200
* Keycloak: 89

Companion services for the PowerGrid use case run on the host (local dev) and are reached by the
containers via `host.docker.internal`:
* PowerGrid simulator (provided example): 5122
* PowerGrid expert agent API: 5123
* A3S (if used instead of the expert agent): 5010

### Authentication data

For a development environment, the system uses predefined initial data for Keycloak setup.
You can find authentication data under config/dev/cab-keycloak

Some examples of credentials:

| username         | password |
| ---------------- | -------- |
| `admin`          | `test`   |
| `powergrid_user` | `test`   |
| `railway_user`   | `test`   |
| `atm_user`       | `test`   |


By default, the system allows the user to be connected only from a single machine. Which means if you try to connect using the same credentials from another machine, you will be disconnected on the first machine. 

# Development

Contributions to the InteractiveAI Assistant Platform are welcome! To contribute, please make sure to use [developer guide](docs/developer-guide.md)

# Docs
A postman collection is under docs/postman_collections.
You can also check the openapi through the URL http://localhost:[Service port]/docs
