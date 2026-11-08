# divider

A PostgreSQL-backed job queue for [Quarz](https://github.com/quarz-lang/quarz),
on [rotor](https://github.com/quarz-lang/rotor): jobs are rows of
`divider_jobs`, inserted in your transactions and run by workers in Quarz
tasks. Named after the frequency divider of a quartz movement, which
turns the crystal's oscillation into the pulses that drive the hands.

Work in progress: the design is
[divider#1](https://github.com/quarz-lang/divider/issues/1). So far there are
the schema (`Divider::Migrations`), job classes and their options, their
arguments (encoded; their decoding waits for quarz-lang/quarz#400, and
the `args` statement itself for quarz-lang/quarz#397), `enqueue`,
`Divider.cancel`, unique jobs and the worker (`Divider::Worker`,
`Divider.serve`). Cron and the testing helpers come next.

```quarz no_run
class Cleanup < Divider::Job
  queue :maintenance
  unique period: 1.hour
end

data WelcomeArgs = (user_id: Int, template: String = "welcome")

class SendWelcome < Divider::Job
  queue :mail
  args WelcomeArgs
  unique by: [:user_id]
end

def nightly(repo: Rotor::Repo) -> Void
  Cleanup.enqueue(repo, after: 10.minutes)
  SendWelcome.enqueue(repo: repo, user_id: 42)
  SendWelcome.enqueue_args(repo: repo, args: WelcomeArgs.new(42, "again"), priority: 1)
end
```

A job with `args` is enqueued with keywords: `repo:`, its arguments
(optional where the `data` type has a default), and `at:` or `after:`,
`priority:`. Those keywords are checked at compile time by their outer
class only (`Type#fields` shows a nullable field as `Object` and an
`Array[Int]` as `Array`: quarz-lang/quarz#401), so a value of the wrong
type there is found when the worker decodes the job (`Divider::ArgsError`,
discarded). `enqueue_args` takes an `Args` value, checked fully. For the
same reason `args` checks that a field can be encoded by its outer class
only: a nullable or union field, or a collection's elements, of a class
that cannot (`Pen?`, `Array[Pen]`), is found at decoding too. A job
with `args` also keeps the argument-less `enqueue(repo, at:, after:,
priority:)` it inherits, which a call naming none of the arguments
(`SendWelcome.enqueue(repo: repo)`) resolves to: it raises
`ArgumentError` ("SendWelcome.enqueue takes its arguments (WelcomeArgs)
as keywords, or a WelcomeArgs value with enqueue_args") when it runs,
before inserting anything. A call that names an argument is checked at
compile time.

## Running jobs

```quarz no_run
class Cleanup < Divider::Job
  def perform(job: Divider::Context) -> Void
    job.repo.exec("DELETE FROM sessions WHERE expires_at < now()")
  end
end

worker = Divider::Worker.new(ENV["DATABASE_URL"], jobs: [Cleanup], queues: {default: 10, maintenance: 1})
Signal.trap(:TERM) { worker.shutdown(25.seconds) }
worker.run                       # or Task.spawn { worker.run } beside an HTTP server
# Divider.serve(url, jobs: ..., queues: ...) does the same as a process's main.
```

- The worker opens its own pool (the queues' concurrency plus 2
  connections) and a connection that listens for `enqueue`'s
  notifications; it also polls every `poll` (1 second) for scheduled jobs.
- `perform` returning ends the job `completed`; `Divider::Discard`
  discards it; `Divider::Snooze.new(10.minutes)` puts it off without
  counting the attempt; `Divider::Cancelled` (after
  `job.cancel_requested?`) cancels it; any other exception, or the class's
  `timeout`, fails the attempt, retried after `attempt ** 4 + 15` seconds
  plus up to 30, until `max_attempts`, then discarded. Each failure is
  recorded in the row's `errors`.
- `shutdown(grace)` stops fetching and gives the running jobs `grace`
  to finish; those still running are cancelled and go back to
  `available`, the attempt not counted.
- Every 30 seconds a node marks its running jobs alive; one node at a
  time (an advisory lock) gives back the running jobs of a node silent
  for 5 minutes and deletes completed jobs after 7 days, discarded and
  cancelled ones after 30.
- A worker fetches only the classes in its `jobs` and never touches the
  others. Every node running a queue should know all of that queue's
  classes: queues are how work is split between nodes of different kinds.
  A row whose class no deploy knows any more stays `available` until
  `Divider.cancel` removes it.
- `on_error:` and `on_event:` (`Divider::Event`: started, completed,
  failed, discarded, snoozed, cancelled, rescued, with durations) receive
  what happens; without `on_error`, errors are printed to standard error.

## License

MIT No Attribution (LICENSE).
