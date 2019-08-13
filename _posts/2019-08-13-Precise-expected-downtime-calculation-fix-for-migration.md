# Precise expected downtime calculation fix for migration:

Expected downtime field means how much time the system needs to send
everything that is pending to finish migration, if this expected downtime
reaches <= downtime-limit Qemu completes the migration. By default,
Qemu uses downtime-limit as 300 millisecond.

However user can configure the downtime-limit by overriding default(300ms)
value in order to complete migration based on rate of page dirtying and rate
at which page transferred during pre-copy. But it was observed that Qemu
calculation for expected downtime is not accurate which would mislead user to
set downtime-limit wrong and will not allow the migration to complete.

__Qemu Monitor:__
```
info migrate_parameters
compress-level: 1
compress-threads: 8
decompress-threads: 2
cpu-throttle-initial: 20
cpu-throttle-increment: 10
tls-creds: ''
tls-hostname: ''
max-bandwidth: 9223372036853727232 bytes/second
downtime-limit: 300 milliseconds
x-checkpoint-delay: 200
block-incremental: off
```
```
info migrate:
Migration status: active
total time: 130874 milliseconds
expected downtime: 1475 milliseconds
setup: 3475 milliseconds
transferred ram: 18197383 kbytes
throughput: 866.83 mbps
remaining ram: 376892 kbytes
total ram: 8388864 kbytes
duplicate: 1678265 pages
skipped: 0 pages
normal: 4536795 pages
normal bytes: 18147180 kbytes
dirty sync count: 6
page size: 4 kbytes
dirty pages rate: 39044 pages
```

If we observe,
```
default downtime-limit: 300 millisecond
expected_downtime: 1475 millisecond (not accurate)
```
Actual calculation:
```
expected_downtime = remaining ram / throughput
expected_downtime = 376892 bytes / 866.83 mbps => 3478.34 (actual value)
```
It was fixed by my [expected downtime calculation fix patch](https://lists.gnu.org/archive/html/qemu-devel/2018-04/msg02417.html) that helps user to migrate by configuring accurate downtime-limit. But got suggestions from community that the calculation is of not accurate and
we need to consider the ram that gets re-dirtied during the time when we would
have actually transferred ram in the current iteration.

Then I did work on [expected
downtime calculation precision patch](https://lists.gnu.org/archive/html/qemu-devel/2019-01/msg05421.html), where I have came up with a calculation by considering the ram that could
get redirtied during the current iteration at the time we would have
transferred the remaining ram in current iteration. By this way,
the total ram to be transferred will be remaining ram + redirtied ram
and dividing with bandwidth would yield us better expected downtime
value.

Later community accepted the existing calculation which I had fixed in previous patch is accurate as, expected downtime is considered as how long time the guest will be down if we stop the VM right now and migrate.
And if we stop the VM then there wont be any re dirtying of pages in
[review comment](https://lists.gnu.org/archive/html/qemu-devel/2019-01/msg06100.html).

This calculation precision was felt practical in a bug where precopy migration for a hugepage backed VM that would continue infinitely even for
idle guest because for migration qemu uses 4K pagesize and if any single page dirtied by guest resulted in (Hugepage size / 4K) pages to be re-transmitted again.

In Power, with 16M hugepage backed POWER8 guest failed to successfully migrate as actual expected downtime was greater than 300 millisecond and
downtime-limit set by user based on expected downtime wrongly calculated was inaccurate and qemu doesn't converge for handing over the VM to destination cause it to migrate infinitely.
