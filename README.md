# zagentcapsule

`zac` is a single-file, zero-dependency Java 25 CLI that runs **any command** inside an isolated [Apple container](https://github.com/apple/container/) with explicit filesystem, network, credential, CPU, and memory capabilities.

It is a supervisor around Apple's `container` CLI that replaces its large, low-level flag surface with a small capability vocabulary, denies by default, and prints the effective capability set before every launch.

Autonomous coding agents are what it was built for — an agent running directly on your machine has everything your machine has — but nothing in the tool is agent-specific. The capsule does not know or care what it launches.

## Install

Requires macOS with [`container`](https://github.com/apple/container/) 1.0+ and a JDK 25 on the `PATH`.

```bash
chmod +x zac
sudo ln -s $(pwd)/zac /usr/local/bin/zac    # symlink for development
# or
sudo cp zac /usr/local/bin/
```

No build tool, no dependencies, no JAR — `zac` runs in Java source-file mode.

## Hello world

The simplest possible capsule — run a Java file on Corretto 25, with nothing granted:

```bash
$ cat App.java
void main() {
    IO.println("Hello from " + System.getProperty("java.version"));
}

$ zac run -image:amazoncorretto:25 -workspace:. -- java App.java
```

```
zac 2026-09-02.1
capsule zac-1e2a256e  profile=sealed
  workspace   /Users/abien/hello -> /workspace  (read-only)
  network     disabled
  credentials disabled
  secrets     none
  ports       none
  limits      cpus=2 memory=2G
  image       amazoncorretto:25
warning: network is disabled - nothing in the capsule can reach the network.
         An AI agent will not reach its model API. Grant with -profile:review or -net.
Hello from 25.0.4.1
```

That is the whole contract. No flags means no capabilities: the directory is mounted read-only at `/workspace`, there is no network interface at all, no credentials, and the capsule is deleted when the command exits. Java's source-file mode compiles in memory, so a read-only workspace is enough — nothing needed to be granted to make this work.

The exit code is `java`'s own, so `zac run … && echo ok` behaves the way you expect.

### The same thing, configured

Everything in that command except the command itself is plumbing, and plumbing belongs in `~/.zac/app.properties`:

```properties
image=amazoncorretto:25
cpus=4
memory=8G
workdir=/workspace
```

That is the maximal configuration — those four keys are all `zac` will read. The invocation collapses to:

```bash
$ zac run -- java App.java
```

```
zac 2026-09-02.1
capsule zac-0637f91b  profile=sealed
  workspace   /Users/abien/hello -> /workspace  (read-only)
  network     disabled
  credentials disabled
  secrets     none
  ports       none
  limits      cpus=4 memory=8G
  image       amazoncorretto:25
warning: network is disabled - nothing in the capsule can reach the network.
         An AI agent will not reach its model API. Grant with -profile:review or -net.
Hello from 25.0.4.1
```

`-workspace` was never needed — it defaults to `.`. So `zac run -- <command>` is the floor: the current directory, read-only, sealed off.

Note what did **not** move into the file. The posture — read-only, no network, no credentials — is still stated in full on every launch, because it comes from the `sealed` default rather than from configuration. Adding `profile=trusted` to the file would change nothing; capabilities are not configurable by design. Config removes the plumbing, never the posture:

```bash
zac run -- java App.java                                   # sealed, and you can see it
zac run -profile:review -secret:ANTHROPIC_API_KEY -- claude -p "review this"
#       ^ every grant is still on the command line, always
```

## Use

```bash
export ZAC_IMAGE=my/agent:latest

# run an agent with network (it needs to reach its model API) but a read-only workspace
zac run -profile:review -secret:ANTHROPIC_API_KEY -workspace:. -- claude -p "review this code"

# let it write
zac run -profile:build -secret:ANTHROPIC_API_KEY -workspace:. -- claude -p "fix the failing test"

# keep a capsule around and exec into it
zac create -name:agent1 -workspace:. -- sleep 3600
zac start agent1
zac exec agent1 -- sh -c "ls /workspace"
zac ls
zac inspect agent1
zac stop agent1 && zac rm agent1
```

Everything after `--` is the command to run, passed through untouched.

### Other things worth capsuling

The capability model is not agent-shaped. Anything you would rather not run with your full user rights fits:

```bash
# a build script from a repo you have not read yet — no network, no credentials
zac run -rw -workspace:. -- ./build.sh

# reproduce a bug on a pinned toolchain, sealed off from the network
zac run -image:golang:1.21 -workspace:. -- go test ./...

# run the code an agent just wrote, without the credentials the agent had
zac run -profile:sealed -workspace:./out -- python main.py

# an npm install you want confined to one directory
zac run -profile:build -workspace:. -- npm ci
```

The last one is the pattern worth naming: `review`/`build` to run the *agent*, `sealed` to run what it *produced*. Different trust levels, same tool.

## Capabilities

Access is granted, never assumed. With no flags the capsule gets a read-only workspace, no network, and no credentials.

| Capability | Grant | Revoke | Effect |
|---|---|---|---|
| workspace write | `-rw` | `-ro` | the mount is read-only unless granted |
| network | `-net` | `-no-net` | no interface at all unless granted |
| credentials | `-ssh` | `-no-ssh` | forwards the host SSH agent socket |
| secrets | `-secret:KEY` | — | inherits one host environment variable **by name** |

`-secret:` takes a variable *name*, never a value — the value is inherited from your environment by `container` itself, so it never appears in the command line or the process list. `-secret:KEY=value` is rejected.

### Ports

`-publish:` forwards a host port into the capsule (`container`'s `-p`), repeatable:

```bash
zac run -net -publish:8080 -- java Server.java              # 127.0.0.1:8080 -> 8080
zac run -net -publish:9000:8080 -- java Server.java         # 127.0.0.1:9000 -> 8080
zac run -net -publish:0.0.0.0:8080:8080/tcp -- java Server.java  # reachable from the LAN
```

A spec without a host address binds to `127.0.0.1`, not to every interface, so a published port is reachable from this machine only unless you name another address. Publishing needs a network interface, so `-publish:` is rejected without `-net` or a profile that grants it. Published ports appear in the summary and in the `zac.ports` label.

Also configurable: `-cpus:`, `-memory:`, `-image:`, `-workdir:`, `-name:`, `-workspace:`.

## Configuration

Optional. `~/.zac/app.properties` sets **resource defaults only**:

```properties
image=amazoncorretto:25
cpus=4
memory=8G
workdir=/workspace
```

Precedence: **flag > `$ZAC_IMAGE` (image only) > configuration file > built-in default.**

Capabilities, profiles and the workspace path are deliberately **not** readable from configuration. A sandbox whose posture comes from invisible file state is not one you can audit by reading the command you typed, and the summary `zac` prints before every launch would no longer be the whole truth. Keys like `profile=trusted` or `network=enabled` in the file are silently inert.

For the same reason `zac` reads only `~/.zac/` and ignores any `app.properties` in the working directory — you normally run `zac` from the very directory you are about to expose, so under `-rw` an agent could otherwise write a config file into its own workspace and change the defaults of the next run.

## Profiles

| Profile | Workspace | Network | Credentials |
|---|---|---|---|
| `sealed` *(default)* | read-only | disabled | disabled |
| `review` | read-only | enabled | disabled |
| `build` | read-write | enabled | disabled |
| `trusted` | read-write | enabled | enabled |

The profile sets the baseline; explicit flags always override it, **wherever they appear** — `-rw -profile:sealed` and `-profile:sealed -rw` are identical. A security posture that depended on argument order would be a bug. Granting and revoking the same capability is an error rather than last-wins.

There is no egress filtering in `container`, so network is binary: off, or full egress. **`sealed` cannot run an agent** — the agent cannot reach its model API. `sealed` is for executing agent-*produced* code; `review` is for running the agent itself. `zac` warns about this whenever network is disabled.

Capabilities are fixed when a capsule is created. `zac exec` rejects capability flags rather than silently ignoring them.

## Why not just use `container` directly

The capability mapping is not a convenience wrapper — some of it is load-bearing correctness:

- **`--mount`, never `-v`.** `container`'s `-v host:ctr:ro` passes its third field through unvalidated, so `:readonly`, `:RO`, or any typo silently yields a **read-write** mount. `zac` emits `--mount type=virtiofs,…,readonly`, which rejects unknown directives instead of failing open.
- **`--network none` must be bare.** `none` is matched against the raw, unparsed argument, so `--network none,mtu=1500` means "a network named none" and fails. Omitting `--network` does *not* disable networking — it attaches the builtin network.
- **No `--` before the command.** `container` declares its trailing arguments `captureForPassthrough`, which would hand a separator straight to your command as `argv[1]`.
- **The home-directory guard.** `zac` refuses to mount `$HOME`, `/`, or any ancestor of `$HOME`, which would defeat the point.

## Exit codes

| Code | Meaning |
|---|---|
| `0` | success |
| `2` | `zac` itself refused or failed |
| anything else | the command's own exit code, propagated |

`zac` never uses `--detach`, which would return `0` immediately and destroy that propagation.

## Auditing

`-dry-run` prints the exact `container` command and exits without contacting the daemon:

```
$ zac run -dry-run -workspace:. -image:alpine -- sh -c "echo hi"
zac 2026-09-02.1
capsule zac-ca7f1dac  profile=sealed
  workspace   /Users/abien/proj -> /workspace  (read-only)
  network     disabled
  credentials disabled
  secrets     none
  ports       none
  limits      cpus=2 memory=2G
  image       alpine
warning: network is disabled - nothing in the capsule can reach the network.
         An AI agent will not reach its model API. Grant with -profile:review or -net.
container run --rm --init --name zac-ca7f1dac \
  --mount type=virtiofs,source=/Users/abien/proj,target=/workspace,readonly \
  --network none --no-dns --cpus 2 --memory 2G -w /workspace \
  --label zac=capsule --label zac.profile=sealed --label zac.posture=ro+nonet+nossh \
  alpine sh -c 'echo hi'
```

Every capsule is named `zac-*` and labelled with the posture it was created under, so `zac ls` and `zac inspect` can report it back.

powered by [airhacks.industries](https://airhacks.industries)