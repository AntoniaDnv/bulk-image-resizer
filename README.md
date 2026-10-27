# Bulk Image Resizer

Final exam project for the **Concurrent Programming** course.

## Description

Bulk Image Resizer is a C# console application that reads all images from a source folder (recursively), resizes them to a chosen target size and writes them to an output folder with the same folder structure and file names. The work is split between several parallel workers, so large folders are processed much faster than with a single thread. A live console dashboard shows what every worker is doing, the waiting queue and the finished results.

## What one job is

One job = **resize one image**: load one image file, scale it to the target size (proportionally, so the original aspect ratio is kept) and save it to the matching path in the output folder.

Example: a 3265 x 4898 image with a target size of 800 x 600 becomes 400 x 600.

## Planned features

- **Folders:** choose a source folder and an output folder.
- **Target size:** pick a preset (480 x 360, 800 x 600, 1280 x 720, 1920 x 1080) or a custom width and height, with a "maintain aspect ratio" on/off option.
- **Parallel workers:** a configurable number of workers (for example 4 or 8) take jobs from a shared queue.
- **Start / Stop / Pause / Resume:** keyboard controls that affect all workers together:
  - `S` – Start
  - `P` – Pause
  - `R` – Resume
  - `X` – Stop
- **Workers view:** a live table with one row per worker showing its state (idle, working, paused) and the file it is resizing right now.
- **Queue view:** the number of images still waiting to be processed.
- **Results view:** a table of finished jobs with the file name, original size, new size, time taken, and status (done / error with the error message).
- **Responsive UI:** the console dashboard is redrawn on its own task, so it never freezes while the workers are busy.

## Shared data and how it is protected

| Shared data | Used by | Protection |
|---|---|---|
| Job queue (images waiting to be resized) | All workers take jobs from it | `lock` with a `Lock` object, so a job is taken by only one worker |
| Results list (done / failed images) | Workers add results, the UI reads them | `lock` with a `Lock` object; the UI reads a copy |
| Worker status table | Each worker updates its own row, the UI reads all rows | `lock` with a `Lock` object |
| Progress counters (processed, failed) | All workers increase them | `Interlocked.Increment` |
| Pause / Stop state | Set by the keyboard controls, checked by all workers | Pause signal (`ManualResetEventSlim`) and `CancellationToken` |

**Possible race condition:** two workers taking the same image from the queue at the same time, so one image is resized twice and another one is skipped. Taking a job (checking that the queue is not empty and dequeuing) happens inside one `lock` block.

**Possible deadlock:** a worker holding the queue lock while waiting for the results lock, while the UI holds them in the opposite order. To avoid it, every lock is held only for a short operation and locks are never nested. Slow work, such as resizing and saving the file, is always done outside of any lock.

## Tech

- C# / .NET console application
- Threads / Tasks, `Lock`, `Interlocked`, `CancellationToken`
- An image library for resizing (planned: ImageSharp)
- A console UI library for the live dashboard (planned: Spectre.Console)
