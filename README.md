# divider

A PostgreSQL-backed job queue for [Quarz](https://github.com/quarz-lang/quarz),
on [rotor](https://github.com/quarz-lang/rotor). Jobs are rows of
`divider_jobs`, inserted in your transactions and run by workers in Quarz
tasks: Oban's and Solid Queue's shape (no Redis). Named after the
frequency divider of a quartz movement, which turns the crystal's
oscillation into the pulses that drive the hands.

```toml
# Cog.toml of an application
[dependencies]
Divider = { git = "https://github.com/quarz-lang/divider", branch = "main" }
```

The API reference is at <https://divider.api.quarz.dev>.

The design is [divider#1](https://github.com/quarz-lang/divider/issues/1),
its comments the decisions taken while it was built.

## The schema

`Divider::Migrations.all` are rotor migrations (`divider_jobs`, and
`divider_cron` for the cron entries): add them to your migrator's list.

```crystal
def migrate(repo: Rotor::Repo, app: Array[Type[Rotor::Migration]]) -> Void
  Rotor::Migrator.new(repo, app + Divider::Migrations.all).migrate
end
```

## Jobs

```crystal
data WelcomeArgs = (user_id: Int, template: String = "welcome")

class SendWelcome < Divider::Job
  queue :mail                    # default :default
  max_attempts 10                # default 20
  timeout 2.minutes              # an attempt longer than that fails
  job_args WelcomeArgs           # the arguments: a data type of the class's own
  unique by: [:user_id], period: 1.hour

  def perform(args: WelcomeArgs, job: Divider::Context) -> Void
    # job.repo: a repository over the worker's pool
  end
end

class Cleanup < Divider::Job
  def perform(job: Divider::Context) -> Void
    job.repo.exec("DELETE FROM sessions WHERE expires_at < now()")
  end
end

def signup(repo: Rotor::Repo) -> Void
  repo.transaction do |tx|
    # ... insert the user ...
    SendWelcome.enqueue(repo: tx, user_id: 42)               # exists once tx commits
    nil
  end
  SendWelcome.enqueue(repo: repo, user_id: 42, after: 10.minutes, priority: 1)
  SendWelcome.enqueue_args(repo: repo, args: WelcomeArgs.new(42, "again"))
  job = Cleanup.enqueue(repo, at: Time.utc(2026, 12, 1))     # Divider::Enqueued(id, state, scheduled_at)
  Divider.cancel(repo, job.id)                               # :cancelled, :requested (running) or nil
end
```

- `enqueue` inserts the row and notifies the workers (`pg_notify`) in one
  statement, in the repository's transaction when it is bound to one: the
  job exists, and a worker hears of it, only once that commits. `at:` is a
  time, `after:` a delay from the database's now; `priority:` 0 (first) to
  9, 5 by default.
- A job with `job_args` is enqueued with keywords: `repo:`, its arguments
  (optional where the data type has a default), and the scheduling ones;
  an argument named `repo`, `at`, `after` or `priority` is refused. Those
  keywords are checked by their outer class only (`Type#fields` shows a
  nullable field as `Object` and `Array[Int]` as `Array`, and a keyword
  for a collection is typed `Object`: quarz-lang/quarz#401),
  and `job_args` checks a field's encodability by its outer class only; a
  value that does not fit is found when the worker decodes the job
  (`Divider::ArgsError`, discarded, the error naming where: a deploy that
  changes an `Args` type should add fields with defaults). `enqueue_args` takes an `Args` value,
  checked fully. The argument-less `enqueue(repo)` a job with `job_args`
  inherits raises `ArgumentError`.
- `unique` keeps one live job (scheduled, available, running, or failed
  with a retry pending) per class, values of the `by` arguments (fields
  without a default), and `period` (by the database's clock); `enqueue`
  gives the live one's `Enqueued` instead of inserting. Under REPEATABLE
  READ or SERIALIZABLE it may raise a serialization failure: retry the
  transaction.
- `perform` returning ends the job `completed`; `Divider::Discard` discards
  it; `Divider::Snooze.new(10.minutes)` puts it off without counting the
  attempt; any other exception, or the `timeout`, fails the attempt,
  recorded in the row's `errors` and retried after `attempt ** 4 + 15`
  seconds plus up to 30 (as Oban), until `max_attempts`, then discarded.
- `Divider.cancel` cancels a scheduled, available or failed job; a running
  one is asked to stop: it sees `job.cancel_requested?` and raises
  `Divider::Cancelled` (the job ends `cancelled`), or finishes anyway
  (`completed`, the request kept in `cancelled_at`).

## Running jobs

```crystal
worker = Divider::Worker.new(ENV["DATABASE_URL"], jobs: [SendWelcome, Cleanup],
                             queues: {default: 10, mail: 5})
Signal.trap(:TERM) { worker.shutdown(25.seconds) }
worker.run      # or Task.spawn { worker.run } beside an HTTP server
# Divider.serve(url, jobs: ..., queues: ...): the same as a process's main (TERM/INT).
```

- The worker opens its own pool (the queues' concurrency plus 2
  connections) and a connection that listens for `enqueue`'s
  notifications, opened again if it is lost; it also polls every `poll`
  (1 second) for scheduled jobs.
- `shutdown(grace)` stops fetching and gives the running jobs `grace` to
  finish; those still running are cancelled and go back to `available`,
  the attempt not counted (or to `cancelled`, when that was asked). A
  second `shutdown` (a second Ctrl-C to `Divider.serve`) cancels them at
  once.
- Every 30 seconds a worker marks its running jobs alive; one worker at a
  time (an advisory lock) gives back the running jobs of a worker silent
  for 5 minutes, and deletes completed jobs after 7 days, discarded and
  cancelled ones after 30 (`keep_completed:`, `keep_discarded:`; 0 deletes
  completed jobs at once).
- A worker fetches only the classes in its `jobs` and never touches the
  others: every worker running a queue should know all of that queue's
  classes, and queues are how work is split between workers of different
  kinds. A row whose class no deploy knows any more stays `available`
  until `Divider.cancel` removes it.
- `on_error:` and `on_event:` (`Divider::Event`: started, completed,
  failed, discarded, snoozed, cancelled, rescued, with durations) receive
  what happens; without `on_error`, errors are printed to standard error.
- Each worker's `node` must differ from the others' running at the same
  time (the default adds a random suffix to `HOSTNAME` and the pid).

## Recurring jobs

```crystal
cron = Divider::Cron.new([Divider::Cron.entry("*/5 * * * *", Cleanup),
                          Divider::Cron.entry("0 3 * * *", SendWelcome, user_id: 1, template: "digest")])
Divider::Worker.new(url, jobs: [Cleanup, SendWelcome], queues: {default: 10, mail: 5}, cron: cron).run
```

Five-field expressions in UTC (`@hourly`, `@daily` and the like too). A
worker checks every 15 seconds, by the database's clock, and each entry
is enqueued once per due minute however many workers check (a worker
enqueues entries whose classes it does not run as well: another worker
of their queue runs them). **A minute no
worker checked (every worker stopped) is not made up afterwards.** An
entry's arguments are keywords, encoded when the entry is made and
checked when the job is decoded.

## Testing

```crystal
class SignupTest < Test::Case
  include Divider::Testing::Assertions

  def test_welcome_is_enqueued
    Divider::Testing.fake do
      signup(repo)
      assert_enqueued(SendWelcome, user_id: 42)   # the arguments given, as JSON
    end
  end

  def test_cleanup_runs
    Divider::Testing.inline(repo) { Cleanup.enqueue(repo) }   # run as it is enqueued
    Divider::Testing.drain(repo, jobs: [Cleanup])             # or: run what is due, once
  end
end
```

- `fake` records enqueues (`Divider::Testing.enqueued`) instead of
  inserting them.
- `inline(repo)` runs each job as it is enqueued, in the test's task; one
  enqueued in a transaction runs at the next enqueue outside one, or when
  the block ends (a rolled-back one never). Jobs run as a worker runs them:
  a `Snooze` leaves the job scheduled. A job enqueued `at:` or `after:` a
  time runs at once too; a unique enqueue that gives back a live job runs
  nothing.
- Under `fake`, `unique` is not applied: each enqueue is recorded.
- `drain(repo, jobs:)` runs the jobs of those classes due now, once.
- `fake` and `inline` set a mode for the whole process: they are not for
  tests run in parallel (`quarz test --parallel`).

## In a bezel application

- Add `Divider::Migrations.all` to the migrations `Bezel.main` runs:
  `migrations: Shop::Migrations.all + Divider::Migrations.all` in
  `src/main.qz`, so that `bezel migrate` creates the tables.
- Run the worker beside the endpoint, in the web process
  (`Task.spawn { worker.run }` before `Bezel.start`, with
  `config.database_url`), or as a process of its own whose main is
  `Divider.serve(url, jobs: [...], queues: {...})`.
- In a controller, enqueue in the request's transaction
  (`conn.repo.transaction { |tx| ...; SendWelcome.enqueue(repo: tx, ...) }`)
  so that the job exists only if the request's work was saved.

## License

MIT No Attribution (LICENSE).
