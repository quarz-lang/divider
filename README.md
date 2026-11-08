# divider

A PostgreSQL-backed job queue for [Quarz](https://github.com/quarz-lang/quarz),
on [rotor](https://github.com/quarz-lang/rotor): jobs are rows of
`divider_jobs`, inserted in your transactions and run by workers in Quarz
tasks. Named after the frequency divider of a quartz movement, which
turns the crystal's oscillation into the pulses that drive the hands.

Work in progress: the design is
[divider#1](https://github.com/quarz-lang/divider/issues/1). So far there are
the schema (`Divider::Migrations`), job classes and their options,
`enqueue`, `Divider.cancel` and unique jobs. Job arguments, the worker,
cron and the testing helpers come next.

```quarz no_run
class Cleanup < Divider::Job
  queue :maintenance
  unique period: 1.hour
end

def nightly(repo: Rotor::Repo) -> Void
  Cleanup.enqueue(repo, after: 10.minutes)
end
```

## License

MIT No Attribution (LICENSE).
