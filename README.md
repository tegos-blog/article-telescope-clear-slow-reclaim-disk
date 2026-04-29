# Why telescope:clear Is Slow and How to Reclaim Disk in Seconds

![Why telescope:clear Is Slow and How to Reclaim Disk in Seconds](assets/poster.jpg)

A while back I wrote about `laravel-telescope-flusher`. It just hit 1,000 installs on Packagist 🎉, so I sat down to back up the original post with real benchmark numbers.

Spoiler: `telescope:clear` takes 2.5 hours and leaves 3 GB locked in your `.ibd` files. `telescope:flush` does the same job in 1.21 seconds and actually gives the disk back.

## What's Inside

- Why `telescope:clear` is slow: chunked `DELETE LIMIT 1000` + FK cascade on `telescope_entries_tags`
- The InnoDB trap: `info_schema` says the table is empty but the `.ibd` file stays huge
- Real benchmark on 1M entries / 3M tags: clear (9025 s, 3.1 GB on disk) vs flush (1.21 s, 428 KB)
- Full source of `telescope:flush`: `TRUNCATE` all 3 tables + `OPTIMIZE TABLE`, with a local-only guard
- Decision table: when to use `clear`, `prune`, or `flush`

## 📎 Read Full

[Why telescope:clear Is Slow and How to Reclaim Disk in Seconds](https://dev.to/tegos/why-telescopeclear-is-slow-and-how-to-reclaim-disk-in-seconds-26of)
