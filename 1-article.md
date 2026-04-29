A while back I wrote about [`laravel-telescope-flusher`](https://dev.to/tegos/efficiently-managing-telescope-entries-with-laravel-telescope-flusher-484a) - a tiny package I built to wipe Telescope data without waiting forever. It just hit **1,000 installs** on Packagist 🎉, so it felt like the right time to actually back up the original post with real numbers, not just claims.

So I sat down, seeded a million Telescope entries on a fresh MySQL 8.0, and timed the three things you'd reach for: `telescope:clear`, `telescope:prune`, and `telescope:flush`. Spoiler: the gap is bigger than I expected.

![Telescope flush vs clear demo](https://raw.githubusercontent.com/tegos/laravel-telescope-flusher/refs/heads/main/assets/flush-demo.gif)

## Quick recap of why `telescope:clear` is slow

Two things kill it. First, the loop:

```php
// vendor/laravel/telescope/src/Storage/DatabaseEntriesRepository.php
public function clear()
{
    do {
        $deleted = $this->table('telescope_entries')->take($this->chunkSize)->delete();
    } while ($deleted !== 0);
    // ...same for telescope_monitoring
}
```

`$chunkSize = 1000`. A million rows = a thousand round-trip `DELETE` statements, each writing to redo log, undo log, double-write buffer.

Second (and this one I missed in the original post): `telescope_entries_tags` has a foreign key on `entry_uuid` with `ON DELETE CASCADE`. With ~3 tags per entry, every parent delete triggers a cascade delete on the tag table. On a million entries, that's **3 million extra deletes** the loop never asked for.

`telescope:prune --hours=24` is the same loop with a `WHERE` filter. Same problem.

## And `DELETE` doesn't give you the disk back

I missed this on the first pass. After `telescope:clear` finishes, `information_schema.tables` reports the data length as basically zero. Looks done. Then check the actual file:

```bash
ls -lah /var/lib/mysql/telescope_test/telescope_*.ibd
```

The `.ibd` files are still huge. InnoDB **doesn't return space to the OS after `DELETE`** - it only marks pages reusable for future inserts. To actually shrink the file you need `OPTIMIZE TABLE` (which rebuilds it) or `ALTER TABLE ... ENGINE=InnoDB`.

`telescope:clear` does neither. So your dev disk stays full.

## The benchmark

Setup: MySQL 8.0 in Docker, default config. Seed: **1,000,000** `telescope_entries` (~2 KB JSON content each), **3,000,000** rows in `telescope_entries_tags`, real foreign key with cascade. Bench script lives in [`bench/`](https://github.com/tegos/laravel-telescope-flusher/tree/main/bench) - go run it yourself.

Starting state, identical for both runs:

```plaintext
telescope_entries          rows=1000000   logical=2.33 GB   .ibd=2.4 GB
telescope_entries_tags     rows=3000000   logical=672 MB    .ibd=688 MB
telescope_monitoring       rows=50        logical=16 KB     .ibd=112 KB
TOTAL                      rows=4000050   logical=2.99 GB   .ibd=3.1 GB
```

Results:

| Step                       | `telescope:clear`      | `telescope:flush` |
|----------------------------|------------------------|-------------------|
| Wall time                  | **9025 s** (≈150 min)  | **1.21 s**        |
| Logical size after         | 128 KB                 | 128 KB            |
| `.ibd` files on disk after | **3.1 GB** (unchanged) | **428 KB**        |

That's roughly **7400× faster** and 3 GB of disk you actually get back. Both runs leave `info_schema` reporting the same size, by the way. That's the trap. Only `ls -lah` on the `.ibd` files tells you the truth.

`prune --hours=0` benches almost identically to `clear` (same loop, same FK cascade), so I didn't bother running it to completion. The shape of the result is the same.

## What `flush` does differently

The package's whole command is short enough to paste:

```php
DB::getSchemaBuilder()->withoutForeignKeyConstraints(function () {
    DB::table('telescope_entries')->truncate();
    DB::table('telescope_entries_tags')->truncate();
    DB::table('telescope_monitoring')->truncate();
});

if (DB::getDriverName() === 'mysql') {
    DB::statement('OPTIMIZE TABLE telescope_entries');
}
```

No magic. `TRUNCATE` is a metadata operation. Instant on InnoDB, no per-row work, no cascade. The `withoutForeignKeyConstraints` wrapper is needed because `TRUNCATE` doesn't fire cascades, so you have to disable the FK check yourself. `OPTIMIZE TABLE` rebuilds the table on `innodb_file_per_table` (the default for years) and produces a fresh, tiny `.ibd`.

There's also an `App::isLocal()` guard - `TRUNCATE` is irreversible, you really don't want to fat-finger this anywhere except dev.

## When to use what

| Approach                     | Use case                                                                                            |
|------------------------------|-----------------------------------------------------------------------------------------------------|
| `telescope:clear`            | Default local cleanup. Works, slow on big tables, leaves disk allocated.                            |
| `telescope:prune --hours=24` | Scheduled retention - keep last N hours. Same disk problem, but table size stays bounded over time. |
| `telescope:flush` (package)  | Dev nuke. Telescope ballooned, you want it gone in a second and the disk back.                      |

I don't run Telescope in production, neither should you, so the local-only guard isn't a limitation.

## TL;DR

- `telescope:clear` = chunked `DELETE LIMIT 1000` + cascading FK on tags. On 1M entries: **2.5 hours**.
- InnoDB **doesn't shrink the `.ibd` after `DELETE`**. `info_schema` lies, `ls -lah` doesn't.
- `telescope:flush` = `TRUNCATE` + `OPTIMIZE TABLE`. **1.21 s** on the same data, 3 GB → 428 KB on disk.
- If your `info_schema` says the table is empty but `df` disagrees, it's the InnoDB pages, not your imagination.

## Resources

- 👉 [Package on GitHub](https://github.com/tegos/laravel-telescope-flusher)
- [Package on Packagist](https://packagist.org/packages/tegos/laravel-telescope-flusher)
- [Original post (the "why" without numbers)](https://dev.to/tegos/efficiently-managing-telescope-entries-with-laravel-telescope-flusher-484a)
- [MySQL docs: `OPTIMIZE TABLE`](https://dev.mysql.com/doc/refman/8.0/en/optimize-table.html)

## Author's Note

Thanks for sticking around!
Find me on [dev.to](https://dev.to/tegos), [linkedin](https://www.linkedin.com/in/ivan-mykhavko/), or you can check out my work on [github](https://github.com/tegos).

**Notes from real-world Laravel.**
