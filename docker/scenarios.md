# 🐳 Docker — Scenario-Based Interview Questions

---

**Q1. [L1] A container exits immediately after starting. How do you debug it?**
**Answer:** `docker logs <container-id>` — read why it exited. Use `docker run -it <image> sh` to start interactively. Check the entrypoint/cmd. Common cause: the main process exits (e.g., a script returns 0), so the container stops. Containers live only as long as their PID 1 runs.

**Q2. [L2] Your Docker build fails with "no space left on device" on the CI server. What do you do?**
**Answer:** Clean up: `docker system prune -af` removes unused images, containers, volumes, build cache. `docker images | grep '<none>'` finds dangling images. Set up automatic cleanup cron: `docker system prune -f --filter "until=24h"`. Long term: add more disk or use BuildKit with a cache limit.

**Q3. [L2] A containerized app can't connect to a database container on the same Docker host. What's wrong?**
**Answer:** By default, containers are isolated. They need to be on the same Docker network. Create a network: `docker network create app-net`. Run both containers with `--network app-net`. Then use the container name as hostname. If using Docker Compose, all services in the same compose file are auto-networked.

**Q4. [L2] Your Docker image is 2GB. How do you reduce it?**
**Answer:** Multi-stage build (build artifact in fat builder image, copy only artifact to slim runtime). Use alpine-based images. Clean up package manager cache in the same RUN layer. Remove build tools and test files. Use `.dockerignore`. Check with `docker history <image>` or `dive <image>` to find large layers.

**Q5. [L2] How do containers in the same pod (Kubernetes) communicate vs containers in different pods?**
**Answer:** Same pod: share network namespace → communicate via `localhost`. Different pods: need a Kubernetes Service. Direct pod IP works but is ephemeral (changes on restart). Always use Service DNS names.

**Q6. [L3] You need to run a Docker container with access to GPU. How?**
**Answer:** Install NVIDIA Container Toolkit on the host. Run with `--gpus all` or `--gpus '"device=0"'`. In Docker Compose: `deploy: resources: reservations: devices:`. The container can then use CUDA APIs. Verify: `docker run --gpus all nvidia/cuda:11.0-base nvidia-smi`.

**Q7. [L2] A container is consuming 100% CPU. How do you limit it without restarting?**
**Answer:** `docker update --cpus="1.5" <container>` — limit to 1.5 CPU cores. Also `--memory="512m"` for memory limit. For Kubernetes, update the resource limits in the pod spec and trigger a rolling restart.

**Q8. [L2] Explain the difference between COPY and ADD in a Dockerfile.**
**Answer:** COPY: copies files from build context to image. Simple and explicit. ADD: same as COPY but also supports URLs and auto-extracts tar archives. Prefer COPY — ADD's extra features make behavior less predictable. Use ADD only when you specifically need tar extraction.

**Q9. [L3] Your container needs to run as a non-root user for security. How do you set this up?**
**Answer:**
```dockerfile
RUN groupadd -r appuser && useradd -r -g appuser appuser
COPY --chown=appuser:appuser . .
USER appuser
```
The `USER` instruction sets the running user. Application files must be owned by this user. In Kubernetes, set `securityContext.runAsNonRoot: true` and `runAsUser: 1000`. Most base images now have a non-root user you can use (e.g., `node` user in Node.js images).

**Q10. [L2] What is the difference between CMD and ENTRYPOINT?**
**Answer:** ENTRYPOINT defines the main command that always runs. CMD provides default arguments. If ENTRYPOINT is `["nginx"]` and CMD is `["-g", "daemon off;"]`, running `docker run myimage -t` would execute `nginx -t` (CMD replaced by args). If only CMD: `docker run myimage bash` runs bash instead. Best practice: use ENTRYPOINT for the executable, CMD for default arguments.

**Q11. [L2] How do you share data between the host and a container?**
**Answer:** Bind mount: `docker run -v /host/path:/container/path`. Volume: `docker run -v myvolume:/container/path` (Docker manages storage location). Use bind mounts for development (live code changes). Use named volumes for production data (better performance, portable). Avoid bind mounts in production — ties container to specific host path.

**Q12. [L3] You have a microservices app with 10 containers. How do you manage them locally?**
**Answer:** Docker Compose. Define all services in `docker-compose.yml`. `docker-compose up` starts everything. Auto-creates networks, manages startup order with `depends_on`, handles volume mounts. For production: Kubernetes. Compose is for local dev only.

**Q13. [L2] A container is running but your app inside it crashed. Docker shows the container as "Up." Why?**
**Answer:** The container's PID 1 (init process) is still running but the app process (a child) crashed. If using shell scripts as entrypoint, the shell is still alive even after the app it started dies. Fix: use `exec` in shell scripts to replace the shell with the app process: `exec node server.js`. Or use a proper init system in the container. Add a healthcheck to detect app failure.

**Q14. [L2] How do you view resource usage (CPU, memory) of running containers?**
**Answer:** `docker stats` — live stream of CPU%, memory, network I/O, block I/O per container. `docker stats --no-stream` for a one-time snapshot. For historical metrics: use cAdvisor + Prometheus for container-level metrics, or cloud-native tools (CloudWatch Container Insights for ECS/EKS).

**Q15. [L3] Your Docker Compose app needs to wait for a database to be ready before starting the app container. How do you implement this?**
**Answer:** `depends_on` only waits for container start, not for the service inside to be ready.

Solutions:
1. **wait-for-it.sh** — shell script that polls the DB port until it responds.
2. **healthcheck + depends_on condition**:
```yaml
db:
  image: postgres
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres"]
    interval: 5s

app:
  depends_on:
    db:
      condition: service_healthy
```
`service_healthy` waits for the healthcheck to pass before starting the app.

**Q16. [L2] What is Docker BuildKit and why is it better than the classic builder?**
**Answer:** BuildKit is the modern Docker build backend. Benefits: parallel layer building (faster), better caching, secret mounts (pass secrets to build without storing in image), SSH mount for private repos, inline cache export, reduced image size. Enable: `DOCKER_BUILDKIT=1 docker build .` or set in Docker daemon config. Default in Docker 23+.

**Q17. [L3] You need to pass a GitHub token to `npm install` during Docker build without it ending up in the image. How?**
**Answer:** BuildKit secret mounts:
```dockerfile
RUN --mount=type=secret,id=github_token \
    export GITHUB_TOKEN=$(cat /run/secrets/github_token) && \
    npm install
```
Build: `docker build --secret id=github_token,src=~/.github_token .`

The secret is available only during that RUN step and NOT stored in any layer.

**Q18. [L2] What is the difference between `docker stop` and `docker kill`?**
**Answer:** `docker stop` sends SIGTERM, waits 10 seconds (configurable), then SIGKILL. App can handle SIGTERM for graceful shutdown. `docker kill` sends SIGKILL immediately (or a specified signal). Use `stop` for normal shutdown. Use `kill` only when `stop` doesn't work.

**Q19. [L2] How do you inspect a running container's environment variables and configuration?**
**Answer:** `docker inspect <container>` — full JSON config including environment variables, mounts, network, etc. `docker exec <container> env` — shows env vars from inside the container. `docker inspect --format='{{.Config.Env}}' <container>` for just env vars.

**Q20. [L3] Explain Docker networking modes.**
**Answer:**
- `bridge` (default) — container gets its own IP on a virtual network. Containers talk via bridge or by name (user-defined networks).
- `host` — container shares the host's network namespace. No port mapping needed. Better performance. Less isolation.
- `none` — no networking. Completely isolated.
- `overlay` — for Docker Swarm. Connects containers across multiple hosts.
- `macvlan` — assigns a real MAC address from the host network. Container appears as a physical device on the network.

---

**Q21-Q60. Rapid-fire Docker Scenarios**

**Q21. [L1]** How do you get a shell inside a running container? **Answer:** `docker exec -it <container> bash` or `sh` if bash isn't available.

**Q22. [L1]** What is the difference between a Docker image and a container? **Answer:** Image = read-only template (blueprint). Container = running instance of an image.

**Q23. [L2]** How do you copy a file from a container to the host? **Answer:** `docker cp <container>:/path/to/file /host/path`.

**Q24. [L2]** Your container image build is failing on `apt-get update`. What's likely wrong? **Answer:** Stale package lists or network issue. Add `--no-cache` to Docker build or restructure to always `apt-get update && apt-get install` in the same RUN command.

**Q25. [L2]** What is a multi-stage build and what problem does it solve? **Answer:** Multiple `FROM` statements in one Dockerfile. Build artifacts in a heavy build stage, copy only what's needed to a slim runtime stage. Final image contains no build tools, reducing size and attack surface.

**Q26. [L2]** How do you set environment variables when running a container? **Answer:** `docker run -e DB_HOST=localhost` or `docker run --env-file .env`. For Compose: `environment:` key or `env_file:` key.

**Q27. [L2]** A container writes logs to a file instead of stdout. How do you collect these logs? **Answer:** Mount a volume and use a log shipper sidecar to read and forward the file. Or configure the app to write to stdout. Containers should write to stdout/stderr — Docker captures these as container logs.

**Q28. [L3]** How do you scan a Docker image for vulnerabilities before pushing to a registry? **Answer:** `trivy image <image>` — scans OS packages and app libraries for CVEs. Integrate in CI: fail the pipeline on HIGH/CRITICAL findings. Also: `docker scout cves` (Docker's built-in tool), `snyk container test`.

**Q29. [L2]** What is the purpose of `.dockerignore`? **Answer:** Excludes files from the Docker build context. Smaller context = faster builds. Prevents accidentally copying secrets, `.git`, `node_modules` into the image.

**Q30. [L3]** How do you implement health checks in Docker? **Answer:** In Dockerfile: `HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost/health || exit 1`. Docker marks container as `healthy` or `unhealthy`. Compose and orchestrators use this for readiness.

**Q31. [L2]** How do you build an image for multiple CPU architectures (AMD64 and ARM64)? **Answer:** Docker Buildx with QEMU emulation: `docker buildx build --platform linux/amd64,linux/arm64 -t myimage:latest --push .` Creates a multi-arch manifest. Users automatically get the right architecture.

**Q32. [L2]** How do you push a Docker image to a private registry? **Answer:** `docker login <registry>`. `docker tag myimage:latest <registry>/myimage:latest`. `docker push <registry>/myimage:latest`.

**Q33. [L2]** What does `EXPOSE` do in a Dockerfile? **Answer:** Documents which port the container listens on. Doesn't actually publish the port. To publish: `docker run -p 8080:3000 myimage`. `EXPOSE` is documentation, not security or networking configuration.

**Q34. [L3]** How do you run Docker Compose in production? **Answer:** Docker Compose is not recommended for production at scale. For production: Kubernetes (EKS/GKE/AKS), ECS, or Docker Swarm (simpler than K8s). Compose is for local development and simple single-host deployments.

**Q35. [L2]** How do you override the default CMD when running a container? **Answer:** Add arguments after the image name: `docker run myimage custom-command --flag`. This replaces CMD. To override ENTRYPOINT: `docker run --entrypoint /bin/bash myimage`.

**Q36. [L2]** What is an `init` container (not Kubernetes)? Why might you use `--init` in Docker? **Answer:** `docker run --init` runs a tiny init process (tini) as PID 1. This properly handles zombie process reaping and signal forwarding. Without init, if PID 1 doesn't forward signals, `docker stop` becomes slow and unreliable.

**Q37. [L3]** How do you debug a container that crashes before you can exec into it? **Answer:** Override the entrypoint to sleep: `docker run --entrypoint sleep myimage 300`. Then exec in and investigate. Or: `docker run --entrypoint /bin/sh -it myimage` to get a shell without running the main app.

**Q38. [L2]** What is image layer caching and why does layer ORDER matter? **Answer:** Docker reuses unchanged layers from cache. Layers that change early in the Dockerfile invalidate all subsequent layers. Put slow-changing layers (base image, dependencies) first. Put fast-changing layers (app code) last.

**Q39. [L2]** How do you see which processes are running inside a container? **Answer:** `docker top <container>` — shows running processes. `docker exec <container> ps aux` for more detail.

**Q40. [L3]** Explain Docker content trust (DCT) and why it matters. **Answer:** DCT allows image publishers to sign images and consumers to verify signatures. `DOCKER_CONTENT_TRUST=1` enforces that only signed images can be pulled/run. Prevents running tampered images. Important for supply chain security.

**Q41. [L2]** What is a "dangling image" in Docker? **Answer:** An untagged image (shows as `<none>:<none>`). Created when you build a new image with the same tag — the old layers lose their tag. Clean up: `docker image prune`.

**Q42. [L2]** How do you set resource limits for a Docker Compose service? **Answer:** Under `deploy.resources.limits`: `cpus: '0.5'` and `memory: 512M`. Note: `deploy` key is only respected by Swarm mode; for regular Compose use `mem_limit` and `cpus` at the service level.

**Q43. [L3]** How does container networking work at the Linux kernel level? **Answer:** Each container gets a network namespace. A virtual ethernet pair (veth) connects the container's namespace to a Linux bridge (docker0). iptables rules handle NAT for outbound traffic and port mapping for inbound. The bridge routes traffic between containers on the same network.

**Q44. [L2]** How do you rebuild only a specific service in Docker Compose? **Answer:** `docker-compose build <service>` then `docker-compose up -d <service>`. Compose won't rebuild services that haven't changed unless you add `--build` flag.

**Q45. [L2]** What is the difference between `docker-compose up` and `docker-compose run`? **Answer:** `up` starts all services defined in the compose file as long-running services. `run` runs a one-off command in a service container: `docker-compose run app pytest`. Useful for running migrations, tests, one-time scripts.

**Q46. [L3]** You need to run a container with access to the host's Unix socket (e.g., Docker socket). What are the security implications? **Answer:** Mounting the Docker socket (`/var/run/docker.sock`) gives the container full control over the Docker daemon — effectively root on the host. Extremely dangerous. Alternative: use Docker socket proxy (Tecnativa) that limits which API calls the container can make. Never mount Docker socket in production unless absolutely necessary.

**Q47. [L2]** How do you roll back to a previous Docker image version in Kubernetes? **Answer:** Update the image tag in the deployment to the previous version's tag. `kubectl set image deployment/app container=registry/app:v1.2.3`. Or `kubectl rollout undo deployment/app` if using Deployment rollout history.

**Q48. [L2]** What is Docker Swarm and how does it compare to Kubernetes? **Answer:** Docker Swarm is Docker's built-in orchestration. Simpler to set up and use than Kubernetes. Less features (no Ingress, limited scheduling, smaller ecosystem). Good for: simple orchestration, small teams, single-cloud. Most production workloads have moved to Kubernetes.

**Q49. [L3]** How do you optimize Docker builds in a monorepo where multiple services share code? **Answer:** Use BuildKit's `--build-context` to pass multiple directories as build context. Use cache mounts for shared dependencies. Or extract shared code to a private package registry (npm, PyPI). Build parent images with shared deps, extend in child service Dockerfiles: `FROM shared-base:latest`.

**Q50. [L2]** How do you make sure containers restart on host reboot? **Answer:** `docker run --restart=unless-stopped` — restarts always except when manually stopped. Or `--restart=always`. Docker Compose: `restart: unless-stopped` in the service definition.

**Q51. [L2]** What is the purpose of `docker commit`? **Answer:** Creates a new image from a running container's current state. Rarely used in production (not reproducible). Use it for: quick debugging snapshots. Never for production images — always use Dockerfiles for reproducibility.

**Q52. [L2]** How do you pass build arguments (not environment variables) to a Docker build? **Answer:** `ARG VERSION=latest` in Dockerfile. Build: `docker build --build-arg VERSION=1.2.3 .`. ARG values are available only at build time, not at runtime (use ENV for runtime). Don't pass secrets via ARG — they appear in image history.

**Q53. [L3]** How do you implement a Docker image garbage collection policy in a registry? **Answer:** ECR lifecycle policies: keep only last N images, delete untagged images after X days. `aws ecr put-lifecycle-policy --lifecycle-policy-text file://policy.json`. For Docker Hub: use retention policies. This prevents registry storage costs from growing indefinitely.

**Q54. [L2]** What does `docker save` and `docker load` do? **Answer:** `docker save myimage > image.tar` — exports image to a tar file. `docker load < image.tar` — imports it. Use for air-gapped environments (no internet, can't pull from registry). Also for shipping specific image versions.

**Q55. [L3]** How do you ensure your containers run with the minimum required Linux capabilities? **Answer:** Drop all capabilities, add only needed ones:
```bash
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myimage
```
In K8s: `securityContext.capabilities.drop: ["ALL"]`, then `add: ["NET_BIND_SERVICE"]`. Running with reduced capabilities limits what a compromised container can do.

**Q56. [L2]** How do you view the Docker build history/layers of an image? **Answer:** `docker history <image>` — shows each layer, its size, and the command that created it. Use `--no-trunc` to see full commands. `dive <image>` for interactive layer explorer.

**Q57. [L2]** What is the difference between `VOLUME` instruction and runtime volume mount? **Answer:** `VOLUME /data` in Dockerfile tells Docker this path should be a volume. Docker auto-creates an anonymous volume at that path if none is provided. Runtime `-v` explicitly mounts a named volume or bind mount. VOLUME instructions ensure data isn't stored in the container layer even if no explicit mount is given.

**Q58. [L3]** How do you run integration tests in CI that require real external services (Redis, Kafka) using Docker? **Answer:** Docker Compose in CI (all services started via compose). Or CI service containers (GitHub Actions services, GitLab CI services). Or testcontainers — a library that programmatically starts Docker containers from test code. Tests spin up the exact services they need, test runs, containers are destroyed.

**Q59. [L2]** How do you debug network connectivity between two containers? **Answer:** `docker network inspect <network>` — see which containers are on the network. From one container: `ping <other-container-name>` (if on same user-defined network). `nslookup <container-name>` to verify DNS. `curl http://<container>:<port>` to test HTTP.

**Q60. [L3]** Explain the security implications of running containers in privileged mode and when it's acceptable. **Answer:** Privileged containers can access all devices, modify kernel parameters, and escape the container namespace. They have near-root access to the host. Acceptable only for: kernel-level tools, CNI plugins, certain monitoring agents, Docker-in-Docker (use Kaniko instead). In production: block with PSA/OPA. Log privileged container creation as a security event.

---

**Q61. [L2] A junior developer complains that every time they edit a file on their host machine, the changes don't physically appear inside the Docker container despite having a bind mount configured. What is the most likely culprit?**

> *What the interviewer is testing:* Inode changing from text editors, bind mount mechanics.

**Answer:**
If they bind-mounted a *single file* (e.g., `-v /path/to/app.py:/app/app.py`) instead of a directory, the issue is how modern text editors (like Vim or some IDEs) save files.
When you save a file in Vim, it often creates a temporary file, deletes the original file, and renames the temp file to the original name. This completely changes the file's **inode**.
Docker bind-mounts are explicitly tied to the inode present at the exact moment the container started. Because the host editor created a new inode, the container continues looking at the old (now deleted/hidden) inode and misses the updates.
*Fix:* Bind-mount the *entire directory* (e.g., `-v /path/to:/app`) rather than the individual file. The directory's inode doesn't change when files inside it are updated.

---

**Q62. [L3] You set up a strict UFW (Uncomplicated Firewall) on your Ubuntu server to block all incoming traffic to port 8080. You then run a Docker container `docker run -p 8080:80 myapp`. Miraculously, a hacker easily accesses your app on port 8080 from the internet. Why did the firewall fail?**

> *What the interviewer is testing:* Docker networking vs. Host iptables/UFW integration.

**Answer:**
Docker bypasses UFW by design.
When Docker starts, it actively manipulates the Linux `iptables` directly by inserting its own rules at the absolute top of the `PREROUTING` chain in the `nat` table (in a chain called `DOCKER`).
Because UFW operates primarily in the `INPUT` chain, the traffic hitting port 8080 is intercepted by Docker's `PREROUTING` rule *before* UFW ever sees it, and routed directly into the container.
*Fix:* Never rely on host OS firewalls to protect exposed Docker ports. You must either not publish the port `8080` to the internet (bind it to localhost `-p 127.0.0.1:8080:80`), or modify the Docker daemon configuration to set `"iptables": false` (which breaks many standard Docker networking features).

---

**Q63. [L1] A container exits with code `137`. What does this specific code universally mean in the Docker ecosystem, and where should you look next?**

> *What the interviewer is testing:* Exit code evaluation, OOM killer.

**Answer:**
Exit code `137` specifically means the container received a `SIGKILL` (signal 9) and was abruptly terminated ($128 + 9 = 137$).
In 90% of Docker/Kubernetes scenarios, this means the container was violently killed by the **OOM (Out Of Memory) Killer** because it exceeded its allocated memory limits.
*Next Steps:* I would immediately run `docker inspect <container_id>` and check the `State.OOMKilled` boolean flag to confirm. Then, I would review application memory profiling and potentially increase the `-m` (memory limit) on the container runtime.

---

**Q64. [L3] You are tasked with debugging a critically failing production container. However, the container is built "Distroless" (it has absolutely no shell, no `bash`, no `ls`, no `curl`). `docker exec` fails with "executable file not found in $PATH". How do you run debugging tools against this container?**

> *What the interviewer is testing:* Namespaces, `nsenter`, ephemeral debug containers.

**Answer:**
You cannot `exec` a shell if the shell binary literally doesn't exist inside the container. You must inject tools from the outside using Linux namespaces.
1. **Find the PID:** Run `docker inspect --format '{{.State.Pid}}' <container_id>`. (e.g., PID 1234).
2. **Use nsenter:** As a root user on the host, use `nsenter` to run a host shell *inside* the network, mount, and PID namespaces of the container:
   `sudo nsenter -t 1234 -n -p -m /bin/bash`
   This gives you full host tools running under the exact perspective of the distroless container.
3. **Alternative (K8s):** Use Ephemeral Containers (`kubectl debug`), which attach a sidecar (like an Alpine/Ubuntu image) sharing the exact same network namespace.

---

**Q65. [L2] A Python data science container parsing massive multi-gigabyte pandas dataframes suddenly crashes randomly. The code is flawless, the server has 128GB of RAM, and OOMKilled is false. You notice the crash happens specifically when multiprocessing writes heavily. What hidden Docker limit is causing this?**

> *What the interviewer is testing:* Shared memory (`shm_size`) limits.

**Answer:**
The container is exhausting its **Shared Memory (`/dev/shm`) limit**.
By default, Docker allocates an incredibly tiny `64MB` to `/dev/shm` for every container. Python multiprocessing, Postgres databases, and tools like Google Chrome Heavily utilize shared memory to pass data quickly between worker processes.
When they try to write a 1GB dataframe into the 64MB shared memory space, they immediately crash with obscure "Bus error" or memory exceptions.
*Fix:* Run the container with an explicitly increased shared memory limit: `docker run --shm-size="2g" myapp`.

---

**Q66. [L1] A developer submits a Dockerfile that copies a 5GB file, runs a command to compress it to 100MB, and then runs `rm` to delete the original 5GB file in the next step. Why does the final Docker image still weigh over 5GB?**

> *What the interviewer is testing:* Intersecting image layers natively, Copy-on-Write storage.

**Answer:**
Dockerfile instructions like `COPY`, `RUN`, and `ADD` create immutable, read-only layers.
When the developer copied the 5GB file in Layer 1, it was permanently baked into the image history. When they deleted it in Layer 3 using a subsequent `RUN rm` command, Docker merely created a new layer with a "whiteout" marker hiding the file. The original 5GB file still exists underneath and is physically downloaded by anyone pulling the image.
*Fix:* Operations that download, process, and delete temporary files must be chained together within a single `RUN` instruction using `&&`:
`RUN wget massive.tar && compress massive.tar && rm massive.tar`

---

**Q67. [L3] Your company is migrating stateful MySQL databases to Docker. A consultant advises using "Bind Mounts". You disagree and advocate strongly for completely bypassing the Docker Storage Driver entirely by utilizing raw Block Devices. Why?**

> *What the interviewer is testing:* Storage drivers under heavy I/O workloads.

**Answer:**
Using standard Docker Storage Drivers (like overlay2) or traversing file-system boundaries for massive, high-IOPS write-heavy database workloads introduces significant systemic overhead. 
While named volumes heavily bypass the UnionFS, for extreme enterprise database performance (bare-metal equivalence), you should allocate a raw LUN or dedicated partition (e.g., `/dev/sdb`) and map it directly into the container using the `--device` flag, allowing the database engine inside the container to interact directly with the kernel's block layer natively, completely eliminating Docker's storage abstraction penalties.

---

**Q68. [L2] You execute `docker run -d myapp`. The terminal returns a long container ID, but immediately upon checking `docker ps`, the container is completely missing. `docker ps -a` shows it exited with code 0. Why did it immediately stop if it didn't error?**

> *What the interviewer is testing:* Daemonizing background processes inside containers.

**Answer:**
A Docker container strictly lives only as long as its primary PID 1 process is running.
If the `CMD` or `ENTRYPOINT` in the Dockerfile starts an application in the background (e.g., executing `service nginx start` or appending an `&` to a script), the script will start the daemon, successfully finish its execution, and return an exit code of `0`. 
Because the foreground script finished, PID 1 terminates, and Docker shuts down the container, killing everything inside it.
*Fix:* You must run the main process in the foreground. Use `nginx -g 'daemon off;'` or execute the application binary explicitly without backgrounding it.

---

**Q69. [L1] What happens if your CI pipeline repeatedly builds the exact same Dockerfile using the `latest` tag and pushes it to an AWS ECR registry every day for a year?**

> *What the interviewer is testing:* Tag mutability, dangling references, registry bloat.

**Answer:**
Using the `latest` tag makes it a **mutable tag**. 
Every time the CI pipeline pushes, AWS ECR will overwrite the `latest` tag to point to the brand new image manifest. The older images from previous days will lose their tag and become **Untagged** (Dangling) images in the registry. 
If no Lifecycle Policy is configured to garbage-collect untagged images, you will accumulate 365 orphaned 1GB images, paying AWS for useless storage bloat. (Also, deploying `latest` in Kubernetes is dangerous as it breaks rollback determinism). Always use Git SHAs or Semantic Versioning tags.

---

**Q70. [L3] You want to pass a highly sensitive API key to a running container. You know not to bake it into the image, so you pass it as an environment variable (`docker run -e SECRET=apikey`). Why is this still arguably a security vulnerability, and what is the better approach?**

> *What the interviewer is testing:* Secret exposure via `docker inspect` and `procfs`.

**Answer:**
Passing secrets via Environment Variables (`-e`) is insecure because:
1. Anyone with access to run `docker inspect <container>` on the host will see the secret in plaintext in the JSON output.
2. If the application crashes, the environment variables are often heavily dumped into the error trace logs.
3. The variables are exposed physically in the kernel via `/proc/<PID>/environ`, accessible by any other running process grouped with the same owner.
*Better Approach:* Use Docker Secrets (if in Swarm) or K8s Secrets, which act as a temporary RAM-disk `tmpfs` layer. The secret is securely mounted as a file (e.g., `/run/secrets/apikey`). The application securely reads the file string into memory once, preventing it from appearing in standard diagnostic dumps.

---

**Q71. [L2] A heavily loaded Nginx container starts rejecting connections citing "Too many open files". You check the host Linux server, and its `ulimit -n` is set to 1,000,000. Why is the container still failing?**

> *What the interviewer is testing:* Container-specific ulimit enforcement.

**Answer:**
Containers do not automatically inherit the `ulimit` settings from the user space of the host operating system. The Docker daemon manages its own default limits for containers (often the conservative 1024 or 4096 standard).
To resolve this, you must explicitly pass the ulimits during the container instantiation:
`docker run --ulimit nofile=65536:65536 mynginx`
Alternatively, you can globally alter the defaults overriding the Docker daemon configuration (`/etc/docker/daemon.json`) so all new containers inherit a more production-ready baseline.

---

**Q72. [L1] You are investigating an incident. How do you find the exact time a container was created, started, and stopped down to the millisecond?**

> *What the interviewer is testing:* `docker inspect` parsing.

**Answer:**
You would use the `docker inspect` command to query the detailed metadata json.
You can parse it cleanly using the go-template format flag:
`docker inspect --format='{{.State.StartedAt}} :: {{.State.FinishedAt}}' <container_id>`. This bypasses manually scrolling through massive JSON outputs.

---

**Q73. [L3] Your CI/CD builds for a massive Go monorepo are taking 20 minutes because every minor code change invalidates the `RUN go mod download` layer, forcing a re-download of gigabytes of packages. How do you optimize the Dockerfile to utilize BuildKit cache mounts and permanently speed this up?**

> *What the interviewer is testing:* Advanced BuildKit features, `--mount=type=cache`.

**Answer:**
You need to use a BuildKit persistent cache mount for the dependency directory. This allows the compiler to share an explicit cache folder across completely different consecutive pipeline builds, independently of the image layer cache.
```dockerfile
# Must enable BuildKit
COPY go.mod go.sum .
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o app
```
Even if `go.mod` is bumped and the layer invalidates, the persistent cache mount still contains 99% of the previously downloaded packages on disk, reducing the 20-minute download to seconds.

---

**Q74. [L2] A developer executes a `docker run --rm -v /home/user/code:/app mynode` to run a script containing `npm install`. When the container finishes, the developer finds that all the new node_modules files placed in their `/home/user/code` folder are owned by `root`. Why, and how do you prevent this?**

> *What the interviewer is testing:* UID/GID matching across namespaces, volume permission issues.

**Answer:**
By default, processes inside the container execute as the `root` user (UID 0). 
When those processes write to a bind-mounted directory, they use UID 0 on the host filesystem. Even if you are a regular user on the host, the files are created strictly by root.
*Fix:* You must tightly align the executing user. Run the container explicitly with your current UID/GID by passing the `--user` flag:
`docker run --rm --user $(id -u):$(id -g) -v $(pwd):/app mynode npm install`
The files will then be generated with the exact identical UID/GID mapping back to the host developer.

---

**Q75. [L1] What happens if you run out of IPs in a Docker bridge network? How many IPs does the default Docker bridge give you by default?**

> *What the interviewer is testing:* `docker0` bridge /16 defaults.

**Answer:**
The default `docker0` bridge network utilizes the `172.17.0.0/16` subnet, which provides approximately 65,534 IP addresses. 
If you miraculously exhaust this or configure a custom network with a smaller `/24` subnet and exhaust it, Docker will vehemently fail to start any new containers on that network, citing an IP Address Allocation error in the daemon logs. You would have to aggressively prune dead containers or recreate the network heavily utilizing a larger CIDR block.

---

**Q76. [L3] You deploy a cluster of 50 identical microservices. To ensure zero drifts, they all pull a massive 1GB initial configuration file from a central S3 bucket immediately upon booting via the `CMD` script. Why is this an anti-pattern in container architecture, and what is the immutable alternative?**

> *What the interviewer is testing:* Immutable infrastructure, startup performance, config-maps.

**Answer:**
This brutally violates the principle of **Immutable Infrastructure** and destroys startup agility.
If the S3 bucket goes down, your containers cannot boot. If you deploy 50 pods simultaneously, you abruptly trigger a 50GB spike of completely duplicate network traffic, severely delaying readiness.
*Alternative:* Small, rapidly changing configurations should be mounted externally at runtime via **Kubernetes ConfigMaps** or Docker Swarm Configs (which use fast local tmpfs). If the 1GB file is structurally static (like a machine learning model), it must be baked directly into the Docker image tightly during the CI/CD build phase. The image then acts as an immutable, instant-booting artifact universally across environments.

---

**Q77. [L2] What is a "Dangling Volume", and how does it happen?**

> *What the interviewer is testing:* Data persistence lifecycle.

**Answer:**
A **Dangling Volume** is an orphaned Docker volume that is no longer attached to any active or stopped container.
It often happens when you delete a container with `docker rm <container>` but fail to include the `-v` flag, which instructs Docker to seamlessly delete associated anonymous volumes. Alternatively, scaling down a stateful set explicitly orphans explicitly named volumes.
Because Docker fundamentally prioritizes data safety, it never deletes volumes aggressively automatically. You must run `docker volume prune` manually to securely flush dangling volumes and recover disk space.

---

**Q78. [L1] Are Docker containers fundamentally "virtual machines"? Defend your answer.**

> *What the interviewer is testing:* Core virtualization vs containerization paradigms.

**Answer:**
No, Docker containers natively are **not** Virtual Machines.
A VM relies on a heavily hardware-level Hypervisor (like VMware or KVM) to run an entirely distinct, massive Guest Operating System (with its own kernel) for every application. 
A Docker container uses OS-level virtualization. It tightly shares the exact single underlying Host OS kernel with other containers. It isolates processes locally using Linux features like `Namespaces` (for isolating visibility of network/PIDs) and `Cgroups` (for capping CPU/Memory limits). Containers are vastly lighter because they don't load a 1GB kernel to simply run a 50MB Python app.

---

**Q79. [L3] You've developed an internal tool specifically for your SRE team using Python. Due to compliance, you must heavily sign all your Docker images cryptographically to prove they originated exclusively from your exact CI/CD server before production will run them. What Docker technology enforces this?**

> *What the interviewer is testing:* Docker Trust, Notary, sigstore/cosign.

**Answer:**
This is historically managed by **Docker Content Trust (DCT)**, effectively backed by the Notary service. By enabling `export DOCKER_CONTENT_TRUST=1`, the Docker client cryptographically signs the image manifest using private keys before pushing. Production nodes strictly configured with DCT enabled will adamantly refuse to pull or run images missing signatures from trusted cryptographic publishers.
Modern approaches strongly lean towards utilizing **Sigstore/Cosign**, which enables keyless signing tied to strict OIDC identities (like GitHub Actions workflows) to sign images seamlessly and generate indisputable transparency logs.

---

**Q80. [L2] A junior engineer asks why they can't effectively run a Windows `.exe` binary inside a standardized Ubuntu Docker container running natively on a Windows 10 host using Docker Desktop. Explain the architecture constraint.**

> *What the interviewer is testing:* Environment isolation, Kernel dependencies vs. Host OS functionality.

**Answer:**
Docker Desktop on Windows heavily masks the truth. To run Linux containers, it actually boots a hidden Linux VM deeply in the background (via WSL2 or Hyper-V). The standard "Ubuntu" container runs atop that actual Linux kernel.
You cannot run a Windows `.exe` heavily inside a Linux container because the `.exe` file format fundamentally requires Windows APIs, Windows DLLs, and a native Windows NT kernel. Containers only bundle user-space libraries; they *strictly share* the actively running bare-metal kernel. To run a `.exe` in a container, you must use **Windows Server Containers**, which use a native Windows kernel and natively wrap the `.exe` perfectly. You cannot cross-pollinate kernels.

---

**Q81. [L3] You are running a sidecar monitoring agent alongside your main application container using Docker Compose. The agent needs to see all the processes running inside the main application container using `ps aux`. By default, it can only see its own processes. How do you solve this?**

> *What the interviewer is testing:* PID namespace sharing between containers.

**Answer:**
By default, every Docker container gets its own isolated PID namespace, meaning each container can only see its own processes (its PID 1 and descendants).
To allow the monitoring sidecar to observe the main application's processes, you must share the PID namespace between the two containers using the `pid` option:
```yaml
services:
  app:
    image: myapp
    container_name: main-app

  monitor:
    image: monitoring-agent
    pid: "service:app"
```
With `pid: "service:app"`, the monitor container joins the PID namespace of the `app` container and can see all of its processes via `ps aux` or `/proc`. In plain Docker CLI: `docker run --pid=container:main-app monitoring-agent`.

---

**Q82. [L2] A developer runs a containerized application that writes millions of tiny temporary cache files during processing. After a few hours, the container crashes with "No space left on device" even though `docker stats` shows plenty of disk available. What is happening?**

> *What the interviewer is testing:* tmpfs mounts, inode exhaustion vs. disk space.

**Answer:**
The container is likely running out of **inodes**, not disk space. The overlay2 filesystem has a finite number of inodes, and millions of small files can exhaust the inode table before consuming significant disk bytes. `df -i` inside the container will confirm this.
*Fix:* For temporary cache files that don't need to persist, use a **tmpfs mount** which stores data entirely in RAM and doesn't consume inodes from the overlay filesystem:
`docker run --tmpfs /app/cache:rw,size=512m myapp`
Alternatively, if the cache needs disk backing, mount a dedicated volume (`-v cache-vol:/app/cache`) that has its own independent inode table separate from the container's root filesystem.

---

**Q83. [L2] Your company has a local Docker registry, a staging registry on GCP Artifact Registry, and a production registry on AWS ECR. A developer complains about constantly running `docker login` and `docker logout` to switch between them. What is the cleaner approach?**

> *What the interviewer is testing:* Docker context and credential helpers.

**Answer:**
Docker stores credentials per registry in `~/.docker/config.json`, and you do NOT need to log out of one registry to use another. Each `docker login <registry-url>` adds a separate credential entry. You can push and pull from multiple registries simultaneously without switching.
However, for managing different Docker *daemon endpoints* (local vs. remote hosts), use **Docker Contexts**:
`docker context create staging --docker "host=tcp://staging-host:2376"` then `docker context use staging`.
For the credential management side, configure **credential helpers** (`docker-credential-ecr-login` for AWS, `docker-credential-gcr` for GCP) in `config.json` so authentication tokens auto-refresh without manual `docker login` at all.

---

**Q84. [L2] A containerized microservice makes HTTP calls to `api.example.com`. It works perfectly when tested locally on a developer laptop, but fails with DNS resolution errors when deployed inside a Docker container on the CI server. The CI server itself can resolve the domain fine. What is wrong?**

> *What the interviewer is testing:* Container DNS resolution, Docker's embedded DNS server.

**Answer:**
Docker containers on user-defined networks use Docker's embedded DNS server (`127.0.0.11`). On the default bridge network, containers inherit the host's `/etc/resolv.conf`. However, if the host uses `127.0.0.53` (systemd-resolved's stub resolver), Docker copies this into the container where it's meaningless because the container cannot reach the host's loopback address.
*Fix:* Explicitly configure DNS for the container or Docker daemon:
- Per container: `docker run --dns 8.8.8.8 myapp`
- Globally in `/etc/docker/daemon.json`: `{"dns": ["8.8.8.8", "8.8.4.4"]}`
- Or switch the CI server's systemd-resolved to expose on a real interface rather than the loopback stub.

---

**Q85. [L3] After months of daily Docker builds, your production server's disk is full. You run `docker system prune -af` and reclaim some space, but the disk is still 90% full. `docker system df` shows minimal usage. Where is the hidden disk consumption?**

> *What the interviewer is testing:* overlay2 layer directory orphans, `/var/lib/docker` internals.

**Answer:**
`docker system prune` only removes objects tracked by the Docker daemon (images, containers, volumes, build cache). If the Docker daemon crashed or was force-killed during container operations, orphaned layer directories can accumulate in `/var/lib/docker/overlay2/` that the daemon no longer tracks.
*Diagnosis:* Run `du -sh /var/lib/docker/overlay2/` and compare with `docker system df`. A large discrepancy confirms orphaned layers.
*Fix:* The safest approach is to stop the Docker daemon (`systemctl stop docker`), back up any critical volumes, and reset the storage entirely (`rm -rf /var/lib/docker`). Restarting Docker recreates the directory structure. Re-pull needed images. For prevention, ensure the Docker daemon shuts down gracefully and monitor `/var/lib/docker` disk usage independently.

---

**Q86. [L3] Your security team mandates that Docker must run without root privileges on all developer workstations. The developers still need full Docker build and run capabilities. How do you achieve this?**

> *What the interviewer is testing:* Rootless Docker mode.

**Answer:**
Use **Rootless Docker**, which runs the Docker daemon and all containers entirely within a user's namespace without requiring root privileges on the host.
Installation: `dockerd-rootless-setuptool.sh install` (ships with Docker 20.10+). The daemon runs as the user's systemd service, stores data under `~/.local/share/docker/`, and maps UIDs using `newuidmap`/`newgidmap` (requires `/etc/subuid` and `/etc/subgid` entries).
*Limitations:* Cannot bind to privileged ports (<1024) without `CAP_NET_BIND_SERVICE`. `--net=host` doesn't work. Overlay networks require kernel 5.11+ with unprivileged overlay support. Some storage drivers may have reduced performance. Despite these trade-offs, rootless Docker satisfies the security mandate while preserving standard build and run workflows.

---

**Q87. [L2] Your CI pipeline runs unit tests inside a Docker build using a multi-stage Dockerfile. If the tests fail, you want to extract the test report (JUnit XML) from the failed build stage. But `docker build` exits with an error and produces no final image. How do you get the test report out?**

> *What the interviewer is testing:* Multi-stage build targets, `--target` flag.

**Answer:**
Use the `--target` flag to build only up to the test stage, and structure the Dockerfile so the test report is generated *before* the assertion that causes the build to fail:
```dockerfile
FROM golang:1.21 AS test
COPY . .
RUN go test -v ./... -count=1 2>&1 | go-junit-report > /report.xml; \
    go test ./... -count=1

FROM alpine AS runtime
COPY --from=test /app/binary /app/binary
```
Build with: `docker build --target test -o type=local,dest=./output .` using BuildKit's `--output` flag. Even if the full build fails, you can separately export just the test stage output. Alternatively, use `docker create` on the test target image and `docker cp` the report out.

---

**Q88. [L2] A production container running an API gateway is performing well but you notice its writable layer is growing by 500MB per day. The application itself doesn't write data to disk intentionally. What is causing the growth and how do you stop it?**

> *What the interviewer is testing:* Container writable layer, log files, and read-only filesystem.

**Answer:**
The application or its runtime is writing files to the container's writable layer (the thin read-write layer on top of the image layers). Common culprits: application logs not sent to stdout, temp files, DNS resolver cache, or library-generated cache files.
*Diagnosis:* Run `docker diff <container>` to see every file added (`A`), changed (`C`), or deleted (`D`) in the writable layer since the container started.
*Fix:* 
1. Redirect all logs to stdout/stderr instead of files.
2. Mount writable paths as volumes or tmpfs so writes bypass the container layer.
3. Run the container with a **read-only root filesystem**: `docker run --read-only --tmpfs /tmp --tmpfs /var/run myapp`. This forces you to explicitly declare every writable path, preventing unexpected layer growth.

---

**Q89. [L3] Your CI pipeline runs Dockerized build jobs that themselves need to build Docker images (Docker-in-Docker). The team currently mounts the host Docker socket (`/var/run/docker.sock`). The security team rejects this. What are the alternatives?**

> *What the interviewer is testing:* Docker-in-Docker alternatives, Kaniko, Buildah.

**Answer:**
Mounting the Docker socket gives the inner container full root-equivalent access to the host. Secure alternatives:
1. **Kaniko** — Google's tool that builds container images from a Dockerfile *inside* a container without requiring a Docker daemon. It executes each Dockerfile command in userspace, produces an OCI image, and pushes directly to a registry. Runs unprivileged. Ideal for Kubernetes-based CI (Tekton, GitLab Runner).
2. **Buildah** — Builds OCI images without a daemon. Can run rootless. Supports Dockerfile syntax and its own native commands.
3. **Docker-in-Docker (dind)** — Run a full Docker daemon inside a privileged container. More secure than socket mounting (isolated daemon), but still requires `--privileged`. Use only when Kaniko/Buildah can't satisfy the use case.
4. **BuildKit with remote builder** — Run BuildKit as a separate service and point `docker buildx` at it remotely: `docker buildx create --driver remote --name mybuilder tcp://buildkit:1234`.

---

**Q90. [L2] A container running Nginx generates enormous log files and eventually fills the disk on the Docker host. You want Docker to automatically handle log rotation without modifying the Nginx configuration. How?**

> *What the interviewer is testing:* Docker logging driver configuration, log rotation.

**Answer:**
Docker's default `json-file` logging driver stores container stdout/stderr as JSON files under `/var/lib/docker/containers/<id>/`. Without limits, these files grow unbounded.
Configure log rotation at the container level:
```bash
docker run --log-opt max-size=50m --log-opt max-file=5 nginx
```
Or set it globally in `/etc/docker/daemon.json`:
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5"
  }
}
```
This rotates logs at 50MB per file and keeps a maximum of 5 rotated files (250MB total per container). For Docker Compose, use the `logging:` key under each service. Note: changing the global config only affects *newly created* containers, not existing ones.

---

**Q91. [L3] You deploy a financial application container and the compliance team requires that the container's filesystem must be completely immutable at runtime — no process should be able to write anywhere except explicitly approved paths. How do you enforce this?**

> *What the interviewer is testing:* Read-only root filesystem, defense-in-depth.

**Answer:**
Use the `--read-only` flag to make the entire root filesystem read-only:
```bash
docker run --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \
  --tmpfs /var/run:rw,size=10m \
  -v logs-vol:/var/log \
  myfinancialapp
```
With `--read-only`, any write attempt to an unmounted path returns a "Read-only file system" error. You explicitly whitelist writable paths using `tmpfs` (ephemeral, in-memory) or named volumes (persistent). The `noexec` and `nosuid` flags on tmpfs add additional hardening.
In Kubernetes, set `readOnlyRootFilesystem: true` in the `securityContext`. Combine this with `allowedHostPaths` in PodSecurityPolicy or a Kyverno/OPA policy to restrict volume mounts. This approach follows the principle of least privilege and prevents attackers from writing backdoor binaries even if they compromise the application.

---

**Q92. [L3] Your team wants to implement live migration of a running Docker container from one host to another without stopping it, similar to VM live migration. Is this possible with Docker? What technology enables it?**

> *What the interviewer is testing:* CRIU (Checkpoint/Restore in Userspace), container migration limitations.

**Answer:**
Docker has experimental support for **checkpoint and restore** using **CRIU (Checkpoint/Restore In Userspace)**. CRIU freezes a running process, serializes its entire state (memory, registers, open files, sockets, timers) to disk, and can restore it later — even on a different host.
Workflow:
1. Checkpoint: `docker checkpoint create <container> checkpoint1`
2. Transfer the checkpoint data and container filesystem to the target host.
3. Restore: `docker start --checkpoint checkpoint1 <container>`

*Limitations:* This feature is experimental and not production-ready. Open network connections break (TCP state doesn't survive cross-host migration). External storage must be shared (e.g., NFS). GPU state, complex IPC, and certain kernel features aren't fully supported. For production workloads, Kubernetes pod rescheduling with graceful shutdown/startup is the practical alternative.

---

**Q93. [L2] A developer has a project with a 10GB `data/` directory containing training datasets. Every `docker build` takes 15 minutes before even executing the first Dockerfile instruction. The Dockerfile doesn't reference the `data/` directory at all. Why is it so slow?**

> *What the interviewer is testing:* Docker build context transfer, `.dockerignore`.

**Answer:**
Before executing any Dockerfile instruction, Docker packages the entire **build context** (the directory passed to `docker build`) and transfers it to the Docker daemon. If the `data/` directory is inside the build context, Docker transfers all 10GB every single build — even though no `COPY` or `ADD` references it.
The "Sending build context to Docker daemon... 10GB" message in the build output confirms this.
*Fix:* Add `data/` to `.dockerignore`:
```
data/
*.csv
*.parquet
__pycache__/
.git/
```
This reduces the build context to only the files the Dockerfile actually needs. Alternatively, restructure the project so the Dockerfile lives in a subdirectory without the data, or use BuildKit's ability to specify individual files via `--build-context`.

---

**Q94. [L2] Your Docker Compose file defines 15 services, but during local development, you only need 4 of them running. Starting all 15 wastes resources and slows down your machine. How do you selectively start subsets of services without maintaining multiple Compose files?**

> *What the interviewer is testing:* Docker Compose profiles.

**Answer:**
Use **Compose Profiles** (introduced in Docker Compose v1.28):
```yaml
services:
  api:
    image: myapi
    # No profile = always starts

  worker:
    image: myworker
    profiles: ["full", "backend"]

  ml-engine:
    image: ml-engine
    profiles: ["full", "ml"]

  monitoring:
    image: grafana
    profiles: ["full", "debug"]
```
Services without a `profiles` key always start. Services with profiles only start when that profile is explicitly activated:
- `docker compose up` — starts only services without profiles (api).
- `docker compose --profile backend up` — starts api + worker.
- `docker compose --profile full up` — starts everything.

This is cleaner than `docker compose up service1 service2` because profiles logically group related services and can be combined.

---

**Q95. [L2] A containerized application needs to call an API server running directly on the Docker host machine (not in a container). Using `localhost` or `127.0.0.1` from inside the container doesn't work. How does the container reach the host?**

> *What the interviewer is testing:* Container-to-host networking, `host.docker.internal`.

**Answer:**
Inside a container, `localhost` refers to the container's own loopback interface, not the host's. The container and host have separate network namespaces.
*Solutions:*
1. **Docker Desktop (Mac/Windows):** Use the special DNS name `host.docker.internal`, which automatically resolves to the host machine's internal IP. Available out of the box.
2. **Docker Engine on Linux (20.10+):** Add `--add-host=host.docker.internal:host-gateway` to the run command. `host-gateway` is a special string that Docker resolves to the host's gateway IP (typically `172.17.0.1`).
3. **Docker Compose:**
```yaml
services:
  app:
    image: myapp
    extra_hosts:
      - "host.docker.internal:host-gateway"
```
4. **`--network=host`** mode eliminates the network namespace boundary entirely, but sacrifices container isolation.

---

**Q96. [L2] Your CI pipeline suddenly starts failing with "toomanyrequests: You have reached your pull rate limit" errors when pulling base images from Docker Hub. What is happening and how do you fix it?**

> *What the interviewer is testing:* Docker Hub rate limits, registry mirrors.

**Answer:**
Docker Hub enforces pull rate limits: anonymous users get 100 pulls per 6 hours per IP, authenticated free users get 200. CI servers sharing a single public IP exhaust this quickly.
*Fixes:*
1. **Authenticate:** `docker login` with a Docker Hub account in CI — doubles the limit and tracks per-account instead of per-IP.
2. **Docker Hub Pro/Team subscription** — removes rate limits entirely.
3. **Registry Mirror/Proxy Cache:** Set up a pull-through cache using a local registry: `docker run -d -e REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io registry:2`. Configure the Docker daemon to use this mirror in `/etc/docker/daemon.json`: `{"registry-mirrors": ["http://localhost:5000"]}`. Subsequent pulls hit the local cache.
4. **Copy base images** to your private registry (ECR/GCR/ACR) and reference them from there. This completely eliminates Docker Hub dependency.

---

**Q97. [L3] You need to verify whether a Docker image tagged `myapp:v2.1.0` in your registry is truly a multi-architecture image that supports both `linux/amd64` and `linux/arm64`, without pulling the entire image. How do you inspect this remotely?**

> *What the interviewer is testing:* OCI image manifests, manifest lists, `docker manifest inspect`.

**Answer:**
Use `docker manifest inspect` to query the registry's manifest list without downloading any image layers:
```bash
docker manifest inspect myregistry.com/myapp:v2.1.0
```
For a multi-arch image, this returns a **manifest list** (also called a "fat manifest") containing multiple entries — one per platform. Each entry specifies the `architecture`, `os`, and a digest pointing to the platform-specific image manifest.
If the image is single-architecture, the command returns a single image manifest with layer digests instead of a list.
You can also use tools like **crane** (from Google's go-containerregistry):
```bash
crane manifest myregistry.com/myapp:v2.1.0 | jq '.manifests[].platform'
```
Or **skopeo**: `skopeo inspect --raw docker://myregistry.com/myapp:v2.1.0 | jq .`
These tools query the registry API directly, never downloading image layers.

---

**Q98. [L2] You run a shell script as the ENTRYPOINT that spawns multiple background worker processes. When you `docker stop` the container, it always takes exactly 10 seconds (the timeout) before stopping, and the workers don't clean up properly. What is the root cause?**

> *What the interviewer is testing:* Signal handling, PID 1 behavior, `exec` in entrypoint scripts.

**Answer:**
When Docker sends `SIGTERM` via `docker stop`, it delivers the signal to PID 1 inside the container. If PID 1 is a shell script (`/bin/sh` or `/bin/bash`), the shell does NOT forward signals to its child processes by default. The shell itself ignores `SIGTERM`, so nothing happens for 10 seconds until Docker sends `SIGKILL`.
*Fixes:*
1. **Use `exec`** to replace the shell with the main process: the last line of your entrypoint script should be `exec ./my-worker` instead of `./my-worker`. This makes the worker PID 1 and it receives `SIGTERM` directly.
2. **Trap signals** in the shell script if you must manage multiple processes:
```bash
#!/bin/bash
trap 'kill $(jobs -p); wait' SIGTERM SIGINT
./worker1 &
./worker2 &
wait
```
3. **Use `--init`** flag (`docker run --init`) to inject `tini` as PID 1, which properly forwards signals and reaps zombie processes.

---

**Q99. [L2] After a container has been running for several days, you want to see exactly what files were added, modified, or deleted inside the container compared to its original image. How do you do this without stopping the container?**

> *What the interviewer is testing:* `docker diff`, container writable layer inspection.

**Answer:**
Use `docker diff <container>` to inspect the container's writable layer against its base image:
```bash
docker diff my-running-container
```
Output uses three markers:
- `A` — Added (file didn't exist in the image)
- `C` — Changed (file was modified)
- `D` — Deleted (file existed in image but was removed)

Example output:
```
C /var/log
A /var/log/app.log
C /tmp
A /tmp/cache-abc123
D /etc/original-config.bak
```
This is invaluable for debugging unexpected disk growth, verifying that containers aren't writing to unexpected locations, and auditing what a compromised container may have modified. Combine with `--read-only` root filesystem to prevent unexpected mutations in the first place.

---

**Q100. [L2] You have two containers on the same Docker network. Container A needs to reach Container B, but Container B's name is a long auto-generated string like `project_backend_service_1`. You want a shorter, more memorable hostname. How do you assign one without renaming the container?**

> *What the interviewer is testing:* Docker network aliases, DNS in user-defined networks.

**Answer:**
Use **network aliases** to assign additional DNS names to a container on a specific network:
```bash
docker network create app-net
docker run -d --network app-net --network-alias backend myapp-backend
docker run -d --network app-net myapp-frontend
```
Now `myapp-frontend` can reach the backend using simply `backend` as the hostname, regardless of the actual container name.
In Docker Compose, network aliases are configured under the `networks` key:
```yaml
services:
  project_backend_service:
    image: mybackend
    networks:
      app-net:
        aliases:
          - backend
          - api
```
Multiple aliases can be assigned, and multiple containers can share the same alias (Docker's embedded DNS round-robins between them — useful for simple load balancing). Aliases are scoped to the network, so the same alias can mean different things on different networks.

---

*More Docker scenarios added periodically. PRs welcome.*
