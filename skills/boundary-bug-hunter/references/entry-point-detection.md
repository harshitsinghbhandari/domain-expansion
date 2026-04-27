# Entry-Point Detection Playbooks

How to discover entry points without configuration, per language and framework. The skill scans manifests and source files in parallel and combines results.

The goal is a list of `(name, file, line, kind)` tuples where `kind` is one of: `cli`, `route`, `event-handler`, `cron-job`, `queue-handler`, `library-export`, `ui-component`, `effect-hook`.

If multiple manifests are present (polyglot repo), apply each playbook and union the results.

---

## TypeScript / JavaScript

### Manifest

`package.json`. Read these keys:

- `bin` — every value is a CLI entry point. Mark `kind: cli`.
- `main`, `module`, `exports` — every distinct file path is a `library-export`. With conditional `exports`, walk every condition (`import`, `require`, `default`, `node`, `browser`).
- `scripts` — informational, not entry points themselves.

### Frameworks (detect by dev-dependency or by file presence)

| Framework | Detection signal | Entry-point patterns |
|-----------|------------------|----------------------|
| Express / Koa / Hapi / Fastify | `express`/`koa`/`@hapi/hapi`/`fastify` in deps | `app.<METHOD>(...)`, `router.<METHOD>(...)`, `app.use(...)` — each registered handler is a `route` |
| Next.js | `next` in deps; `pages/` or `app/` dir | Every file in `pages/api/**/*.{ts,js}` is a `route`. `pages/**/*.{tsx,jsx}` and `app/**/page.{tsx,jsx}` are `ui-component` routes. `middleware.{ts,js}` is a `route`. |
| NestJS | `@nestjs/core` in deps | Classes decorated `@Controller(...)` with `@Get/@Post/@Put/@Patch/@Delete` methods → `route`. `@MessagePattern`/`@EventPattern` → `event-handler`. `@Cron` → `cron-job`. |
| Remix / SvelteKit / Astro | `@remix-run/node`, `@sveltejs/kit`, `astro` | Files in `app/routes/`, `src/routes/`, `src/pages/` — each a `route`. |
| AWS Lambda | `aws-lambda` types or `serverless.yml` | Exported `handler` from each function file → `route`. |
| tRPC | `@trpc/server` | Each `.query`/`.mutation`/`.subscription` on a router → `route`. |
| GraphQL (Apollo, Yoga, Mercurius) | `apollo-server*`, `graphql-yoga`, `mercurius` | Each resolver function in the resolver map → `route`. |
| BullMQ / Bee / Agenda | `bullmq`, `bee-queue`, `agenda` | `new Worker(...)`, `queue.process(...)`, `agenda.define(...)` → `queue-handler`. |
| node-cron / agenda cron | `node-cron`, `agenda` | `cron.schedule(...)`, `agenda.every(...)` → `cron-job`. |
| React | `react` in deps | Exported components (function returns JSX, or `forwardRef`/`memo` wrappers) → `ui-component`. Top-level `useEffect`, `useLayoutEffect` calls inside an exported component → `effect-hook`. |
| Vue | `vue` in deps | Exported `defineComponent` / SFC `<script setup>` files → `ui-component`. |

### Generic JS/TS fallback

Every `export default function`, `export function`, and `export const` in a file referenced by `main`/`exports`/`bin` is a `library-export`.

---

## Python

### Manifest

`pyproject.toml` and/or `setup.py`/`setup.cfg`. Read:

- `[project.scripts]` and `[project.gui-scripts]` in `pyproject.toml` → CLI entry points.
- `entry_points={"console_scripts": [...]}` in `setup.py` → CLI entry points.
- `[project.entry-points."<group>"]` (e.g., `pytest11`, `flake8.extension`) → `library-export`.

### Frameworks

| Framework | Detection signal | Entry-point patterns |
|-----------|------------------|----------------------|
| Flask | `flask` in deps | `@app.route(...)`, `@blueprint.route(...)` → `route`. |
| Django | `django` in deps; `urls.py` files | Every `path(...)` and `re_path(...)` → `route`. Management commands (`management/commands/*.py` with `class Command(BaseCommand)`) → `cli`. Signals in `signals.py` registered via `@receiver` → `event-handler`. |
| FastAPI | `fastapi` in deps | `@app.<METHOD>` and `@router.<METHOD>` → `route`. `@app.on_event` (legacy) and `lifespan` async context → startup/shutdown handlers (treat as `event-handler`). |
| Starlette | `starlette` in deps | `Route(...)` and `WebSocketRoute(...)` in route lists → `route`. |
| Click | `click` in deps | `@click.command()` and `@click.group()` decorated functions → `cli`. |
| Typer | `typer` in deps | `@app.command()` → `cli`. |
| Celery | `celery` in deps | `@app.task` / `@shared_task` → `queue-handler`. `app.conf.beat_schedule` entries → `cron-job`. |
| RQ | `rq` in deps | Functions referenced in `queue.enqueue(...)` calls — surface the function definitions as `queue-handler`. |
| AWS Lambda | `aws_lambda_powertools` or `chalice` | `def lambda_handler(event, context)` and Chalice `@app.route` / `@app.schedule` / `@app.on_s3_event` → `route` / `cron-job` / `event-handler`. |
| APScheduler | `apscheduler` in deps | `scheduler.add_job(...)` and `@scheduler.scheduled_job` → `cron-job`. |

### Generic Python fallback

`if __name__ == "__main__":` blocks and any `def main()` referenced from a script entry point.

---

## Go

### Manifest

`go.mod`. The module name plus `package main` declarations identify binaries. Look for:

- Every `package main` with `func main()` → `cli` (or whatever the binary is — server, worker, tool).
- `cmd/<name>/main.go` is the canonical layout; surface each `main` separately.

### Frameworks

| Framework | Detection signal | Entry-point patterns |
|-----------|------------------|----------------------|
| net/http (stdlib) | `import "net/http"` | `http.HandleFunc(...)`, `mux.HandleFunc(...)`, `mux.Handle(...)` → `route`. |
| Gin | `github.com/gin-gonic/gin` | `r.<METHOD>(...)`, `r.Group(...)` chains → `route`. |
| Echo | `github.com/labstack/echo` | `e.<METHOD>(...)` → `route`. |
| Chi | `github.com/go-chi/chi` | `r.<METHOD>(...)`, `r.Mount(...)` → `route`. |
| Cobra | `github.com/spf13/cobra` | `&cobra.Command{Use: "...", Run: ...}` → `cli`. |
| Kong/urfave/cli | similar | Their command struct definitions → `cli`. |
| gRPC | `google.golang.org/grpc` | Generated `RegisterXxxServer` calls — each method on the server impl → `route`. |
| Asynq / River | `github.com/hibiken/asynq`, `github.com/riverqueue/river` | `mux.HandleFunc(...)`, registered worker types → `queue-handler`. |
| Cron | `github.com/robfig/cron` | `c.AddFunc(...)`, `c.AddJob(...)` → `cron-job`. |

### Generic Go fallback

Exported (capital-letter) functions in non-`main` packages → `library-export`.

---

## Rust

### Manifest

`Cargo.toml`. Read:

- `[[bin]]` blocks → each is a `cli` entry point. Default `src/main.rs` is implicit.
- `[lib]` block (or default `src/lib.rs`) → public `pub fn` items are `library-export`s.
- `[[example]]` and `[[test]]` are not entry points but are useful flow starts during analysis.

### Frameworks

| Framework | Detection signal | Entry-point patterns |
|-----------|------------------|----------------------|
| axum | `axum` in deps | `Router::new().route(...)` and `.<METHOD>(...)` chains → `route`. |
| actix-web | `actix-web` | `App::new().route(...)`, `.service(...)` and `#[get]`/`#[post]` macros → `route`. |
| Rocket | `rocket` | `#[get(...)]`, `#[post(...)]`, etc. attribute macros → `route`. |
| Warp | `warp` | `warp::path!(...).map(...)` filters → `route`. |
| Tower | `tower` | `Service` impls registered into a server → `route`. |
| clap | `clap` | `#[derive(Parser)]` structs and `#[command(...)]` → `cli`. |

### Generic Rust fallback

`pub fn` and `pub async fn` items in `lib.rs` and modules re-exported there.

---

## Ruby

### Manifest

`Gemfile` and `*.gemspec`. The gemspec's `executables` field → `cli`. The gem's `lib/<name>.rb` is the library entry.

### Frameworks

| Framework | Detection signal | Entry-point patterns |
|-----------|------------------|----------------------|
| Rails | `rails` in `Gemfile` | `config/routes.rb` — every `get`/`post`/`resources` → `route`. `app/jobs/*.rb` (`< ApplicationJob`) → `queue-handler`. `config/initializers/` register handlers. `app/channels/*.rb` → `event-handler`. |
| Sinatra | `sinatra` in `Gemfile` | `get`/`post`/`put`/`delete` blocks → `route`. |
| Hanami | `hanami` | Action classes — each `call` is a `route`. |
| Sidekiq / Resque / DelayedJob | corresponding gems | Worker classes (`include Sidekiq::Worker`, `extend Resque`) → `queue-handler`. |
| whenever | `whenever` | `config/schedule.rb` entries → `cron-job`. |
| Rake | always available | Tasks in `Rakefile` / `lib/tasks/*.rake` → `cli`. |
| Thor | `thor` | `desc ... ; def <name>` inside `Thor` subclasses → `cli`. |

---

## Java / Kotlin

### Manifest

`pom.xml` (Maven) or `build.gradle` / `build.gradle.kts` (Gradle). Read:

- `mainClass` (Maven) / `application { mainClass }` (Gradle) → `cli`.
- For library projects, public top-level classes and methods → `library-export`.

### Frameworks

| Framework | Detection signal | Entry-point patterns |
|-----------|------------------|----------------------|
| Spring (Boot/MVC/WebFlux) | `spring-boot-starter*`, `spring-web` | `@RestController` / `@Controller` classes — each `@GetMapping`/`@PostMapping`/etc. method → `route`. `@SpringBootApplication` class → `cli`. `@Scheduled` → `cron-job`. `@KafkaListener`/`@RabbitListener`/`@JmsListener` → `queue-handler`. `@EventListener` → `event-handler`. |
| Quarkus | `quarkus-*` deps | `@Path(...)` classes with JAX-RS method annotations → `route`. `@Scheduled` → `cron-job`. |
| Micronaut | `micronaut-*` deps | `@Controller` with `@Get`/`@Post` → `route`. `@Scheduled` → `cron-job`. |
| Ktor | `io.ktor:*` | `routing { get(...) ; post(...) }` blocks → `route`. |
| Vert.x | `io.vertx:*` | `Router::route` registrations → `route`. |

---

## .NET (C#, F#, VB)

### Manifest

`*.csproj` / `*.fsproj` / `*.vbproj`. Read:

- `<OutputType>Exe</OutputType>` → `cli` entry point. The `Main` method is in `Program.cs` (or top-level statements).
- `<OutputType>Library</OutputType>` → `public` types and methods → `library-export`.

### Frameworks

| Framework | Detection signal | Entry-point patterns |
|-----------|------------------|----------------------|
| ASP.NET Core (MVC / Web API) | `Microsoft.AspNetCore.*` | Classes derived from `ControllerBase` / `Controller` — each `[HttpGet]`/`[HttpPost]` method → `route`. Minimal API: `app.MapGet(...)`, `app.MapPost(...)` → `route`. |
| ASP.NET Core SignalR | `Microsoft.AspNetCore.SignalR` | Hub classes — each public method → `event-handler`. |
| Hangfire | `Hangfire.*` | `RecurringJob.AddOrUpdate(...)` → `cron-job`. `BackgroundJob.Enqueue(...)` referenced methods → `queue-handler`. |
| Quartz.NET | `Quartz` | `IJob` implementations → `cron-job`. |
| MassTransit / NServiceBus | corresponding deps | Consumer/handler classes → `queue-handler`. |
| MediatR | `MediatR` | `IRequestHandler<TRequest, TResponse>` impls → `event-handler`. |

---

## PHP

### Manifest

`composer.json`. Read:

- `bin` array → `cli`.
- `autoload.psr-4` namespaces → public classes are `library-export`.

### Frameworks

| Framework | Detection signal | Entry-point patterns |
|-----------|------------------|----------------------|
| Laravel | `laravel/framework` | `routes/web.php`, `routes/api.php` → `route`. `app/Console/Commands/*` → `cli`. `app/Jobs/*` → `queue-handler`. Listeners in `EventServiceProvider` → `event-handler`. Scheduled tasks in `app/Console/Kernel.php#schedule()` → `cron-job`. |
| Symfony | `symfony/framework-bundle` | `#[Route(...)]` attributes on controllers → `route`. `Command` classes → `cli`. Message handlers (`#[AsMessageHandler]`) → `queue-handler`. |
| Slim | `slim/slim` | `$app->get(...)`, `$app->post(...)` → `route`. |

---

## Elixir / Erlang

### Manifest

`mix.exs`. Look for:

- `escript` config → `cli`.
- Library modules with `@spec` and `def` (not `defp`) → `library-export`.

### Frameworks

| Framework | Detection signal | Entry-point patterns |
|-----------|------------------|----------------------|
| Phoenix | `:phoenix` in deps | `lib/<app>_web/router.ex` → `route`. `Phoenix.Channel` modules → `event-handler`. |
| Plug | `:plug` | `Plug.Router` and `match`/`get`/`post` → `route`. |
| Oban | `:oban` | Worker modules (`use Oban.Worker`) → `queue-handler`. Cron jobs in Oban config → `cron-job`. |
| GenServer | stdlib | `handle_call`, `handle_cast`, `handle_info` callbacks in OTP processes — surface as `event-handler` when the process is a long-running supervised service. |

---

## Dart / Flutter

### Manifest

`pubspec.yaml`. Read:

- `executables:` → `cli`.

### Frameworks

| Framework | Detection signal | Entry-point patterns |
|-----------|------------------|----------------------|
| Flutter | `flutter:` in `pubspec.yaml` | `main()` in `lib/main.dart` → `cli`. `Widget build(...)` methods on `StatelessWidget`/`StatefulWidget` → `ui-component`. |
| Shelf | `shelf` | `Router()..get(...)..post(...)` → `route`. |

---

## Generic patterns (any language)

Apply on top of the language-specific playbook:

- **Cron / scheduled jobs**: anything that registers a callback on a time interval → `cron-job`.
- **Queue / pub-sub subscribers**: anything that subscribes to a topic / channel / queue → `queue-handler`.
- **Webhooks**: any HTTP route prefixed `/webhook`, `/hooks`, `/callbacks` → `route` with extra-high suspicion of contract drift (the producer is external).
- **Test entry points (anti-signal)**: existing test files that import or reference the changed code. Use these to find which flows are *already* exercised — they mark verified boundaries.

If after all of the above the agent finds zero entry points but the diff clearly contains code, ask the user which file(s) are entry points and cache the answer in `.boundary-bug-hunter.json`.
