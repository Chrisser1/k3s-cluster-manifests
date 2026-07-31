# Family photos: Nextcloud + Immich on one shared volume

Nextcloud and Immich both read the *same files on disk*. Nothing is synced or
duplicated between them: `family-photos-shared` (a Longhorn RWX PVC, see
`apps/family-photos/`) is mounted read-write into Nextcloud and **read-only**
into Immich, both at `/mnt/family-photos`.

Because Kubernetes PVCs are namespace-scoped — and Longhorn RWX volumes become
unstable when referenced by separate PV/PVC pairs across namespaces — Immich is
deployed into the `nextcloud` namespace rather than its own.

- **Nextcloud** is the drive: upload, auto-upload from phones, bulk imports
  (e.g. rclone from OneDrive), sharing with people outside the family.
- **Immich** is the viewer: timeline, map, faces, semantic search. It indexes the
  same folders via its *External Libraries* feature and never owns the originals.

## Layout on the volume

```
/mnt/family-photos/
  Thomsen/
    Chris/          # one private folder per person
    Steen/
    Anette/
    Carmen/
    _Shared/        # whole-family photos
```

**Immich is opt-in per family.** Only the Thomsen family uses it, so only
`Thomsen/` exists on this volume. The Roer family (`Thomas Skou Roer`,
`ebberoer`) uses plain Nextcloud — their photos stay in their normal Nextcloud
home storage and never touch this volume.

That distinction matters: the Immich pod mounts the *whole* volume read-only, so
anything placed here is readable by the Immich process whether or not a library
points at it. Keeping a family's data off this volume is the real guarantee —
not merely declining to create a library for them. If a family wants a shared
folder but not Immich, use a normal Nextcloud share to their `family-*` group
instead of an external mount.

## How Nextcloud exposes it

The `files_external` app is enabled, and each folder is a **Local** external
storage mount, scoped to a single user or a single family group:

| Mount point | Backing path | Applicable to |
|---|---|---|
| `Photos` | `Thomsen/Chris` | user `Chris` |
| `Photos` | `Thomsen/Steen` | user `Steen` |
| `Photos` | `Thomsen/Anette` | user `Anette` |
| `Photos` | `Thomsen/Carmen` | user `Carmen` |
| `Family Photos` | `Thomsen/_Shared` | group `family-thomsen` |

Everyone sees the same two folder names; the folder each resolves to differs per
user.

> **Never name a mount after a folder the user already has.** Nextcloud mounts
> external storage *over* an existing folder of the same name, so a mount called
> `Photos` silently hides a real `Photos` folder — the files are untouched on
> disk but vanish from the UI, which looks exactly like data loss.
>
> This is not hypothetical: **Nextcloud's default skeleton creates a `Photos`
> folder for every new user** (`/var/www/html/core/skeleton/`). So every account
> has one. Check before creating a mount:
>
> ```bash
> kubectl -n nextcloud exec $NCPOD -c nextcloud -- ls /var/www/html/data/<uid>/files/
> ```
>
> The mount is nevertheless called `Photos`, because that is what it obviously is
> to the people using it. That is only safe because the colliding folder is
> **removed first** — see step 3 of *Adding a new person*. Do not skip that and
> hope the mount "wins"; it will hide the folder instead.
>
> (These mounts were briefly named `Immich` to dodge the collision. That worked,
> but nobody could find their photos, which is its own kind of broken.)

> External Storage was chosen over Groupfolders deliberately: Groupfolders keeps
> data inside Nextcloud's own opaque internal directory structure, which Immich
> cannot read directly. Local external storage points straight at the real path,
> so both apps see identical human-readable folders.

## Auto-upload: use the Nextcloud app, not Immich's

There are two different auto-uploads and they do **not** behave the same:

| App | Photos land in | In Nextcloud | In Immich |
|---|---|---|---|
| **Nextcloud app**, target = `Photos` folder | shared volume | yes | yes |
| **Immich app** backup | Immich's own `/data/upload` | **no** | yes |

Immich's mount is read-only, so its own backup physically cannot write to the
shared volume — those photos land in Immich-owned storage that Nextcloud cannot
see, breaking the single-copy model.

**Tell each person: install the Nextcloud app, enable auto-upload, set the target
to their `Photos` folder.** Only choose Immich's own backup for someone who does
not care about their photos being in the family drive.

## Adding a new person

```bash
NCPOD=$(kubectl -n nextcloud get pod -l app.kubernetes.io/name=nextcloud -o jsonpath='{.items[0].metadata.name}')
alias occ="kubectl -n nextcloud exec $NCPOD -c nextcloud -- php occ"
```

1. **Create the account** (admin UI, or `occ user:add --display-name "Name" uid`).

2. **Add them to their family group** — this alone grants `Family Photos`:

   ```bash
   occ group:adduser family-thomsen theiruid
   ```

3. **Clear the colliding skeleton folder.** Every new account is seeded with an
   empty `Photos` folder, and the mount is also called `Photos` — leaving it in
   place means the mount hides it. The guard matters: if the person has already
   put files there, stop and migrate them (below) instead of deleting:

   ```bash
   kubectl -n nextcloud exec $NCPOD -c nextcloud -- sh -c '
     D=/var/www/html/data/theiruid/files/Photos
     C=$(find "$D" -type f 2>/dev/null | wc -l)
     if [ "$C" -eq 0 ]; then rmdir "$D" && echo "removed empty Photos"; else echo "STOP: $C files present"; fi'
   ```

4. **Create their private folder**, owned by `www-data` (uid 33) so the Nextcloud
   web process can write. Immich runs as root and reads regardless of ownership:

   ```bash
   kubectl -n nextcloud exec $NCPOD -c nextcloud -- sh -c '
     mkdir -p /mnt/family-photos/Thomsen/TheirName &&
     chown 33:33 /mnt/family-photos/Thomsen/TheirName &&
     chmod 755 /mnt/family-photos/Thomsen/TheirName'
   ```

5. **Create the mount and scope it to just them.** The `applicable` step is not
   optional — a freshly created system mount is applicable to *every* user until
   you restrict it:

   ```bash
   occ files_external:create "Photos" local null::null \
     -c datadir=/mnt/family-photos/Thomsen/TheirName
   # note the returned id, then:
   occ files_external:applicable <id> --add-user theiruid
   occ files_external:verify <id>          # expect: status: ok
   ```

   There is **no rename command** for mounts — changing a mount point means
   `files_external:delete` and recreate, then `occ files:scan <uid>`. Deleting a
   mount never touches the files on disk.

6. **Verify no mount is globally visible** before handing over the account:

   ```bash
   occ files_external:list --output=json | jq -r \
     '[.[] | select((.applicable_users|length)==0 and (.applicable_groups|length)==0)]
      | if length==0 then "OK - all mounts scoped" else "WARNING: \(length) global mount(s)" end'
   ```

7. **Add them to Immich** if they want it (see below).

### Migrating existing photos

Photos already in someone's Nextcloud home are in Nextcloud's *internal* storage
and **invisible to Immich**. They must be copied onto the shared volume. Copy
first, verify with checksums, and only delete the originals once both apps show
the photos:

```bash
SRC=/var/www/html/data/theiruid/files/Photos
DST=/mnt/family-photos/Thomsen/TheirName
kubectl -n nextcloud exec $NCPOD -c nextcloud -- sh -c "
  cd $SRC && find . -type f -print0 | xargs -0 md5sum | sort -k2 > /tmp/src.txt
  rsync -a $SRC/ $DST/ && chown -R 33:33 $DST
  cd $DST && find . -type f -print0 | xargs -0 md5sum | sort -k2 > /tmp/dst.txt
  diff -q /tmp/src.txt /tmp/dst.txt && echo VERIFIED_IDENTICAL"
occ files:scan --path="theiruid/files/Photos"
```

### Verifying isolation

Config looking right is not proof. This shows which backing folder each user's
mount actually resolves to (rows appear after the user's first login or scan):

```bash
DBPW=$(kubectl -n nextcloud get secret nextcloud-db-credentials -o jsonpath='{.data.mariadb-password}' | base64 -d)
kubectl -n nextcloud exec nextcloud-mariadb-0 -c mariadb -- mysql -u nextcloud -p"$DBPW" nextcloud -N -e "
  SELECT m.user_id, m.mount_point, s.id FROM oc_mounts m
  JOIN oc_storages s ON s.numeric_id=m.storage_id
  WHERE s.id LIKE '%family-photos%' ORDER BY m.user_id, m.mount_point;"
```

## Adding a person to Immich

Immich accounts are separate from Nextcloud accounts — different password, no
single sign-on.

1. Create the account: `https://photos.hivemindcloud.dk/admin/user-management`
2. Create the library: `https://photos.hivemindcloud.dk/admin/library-management`
   (Avatar → Administration → **External Libraries**). This is a different page
   from *Settings → External Library*, which only holds the global watch and
   scan-schedule options — there is no "create" button there.
   - **Create Library**, pick the owner. **The owner cannot be changed later.**
   - **Folders → Add** → `/mnt/family-photos/Thomsen/<Person>`. Use the path as
     the *container* sees it, which is what is written here — not a host path.
   - **Scan**.
3. Immich indexes in place and never writes to the volume — its mount is
   read-only, so originals cannot be modified or deleted by Immich.

**A library has exactly one owner.** There is no way to point one library at a
parent folder and have each subfolder land in a different person's account —
which is why there is one library per person rather than a single family-wide
one. Sharing between family members is done with Immich albums.

### The shared folder

`Thomsen/_Shared` has a single library owned by `Chris`, rather than one per
person — indexing it four times would duplicate thumbnails and face processing
for identical files. Who owns it barely matters because everyone sees it through
partner sharing (below).

People are scoped per user (`person.ownerId` is `NOT NULL`), not per library, so
because Chris owns both his personal library and `_Shared`, face clustering pools
across both: someone appearing in personal *and* shared photos becomes one person
record, which improves recognition. The trade-off is that those named people
belong to Chris — other family members see the photos but not his people tags,
and name people independently in their own libraries.

## Sharing between family members: partner sharing

Each person keeps their own library and folder; visibility is granted with
**partner sharing**, so photos stay attributed to whoever owns them — Carmen's
appear under Carmen, not pooled into one account.

Set up at `/user-settings` → **Partner Sharing**. Each person does this from
their *own* account; it is not an admin action. Partnerships are **one-way**
(`partner.sharedById` → `sharedWithId`), so mutual visibility needs both
directions — four people all sharing with each other is 12 one-way shares.
`inTimeline` (default `false`) controls whether a partner's photos merge into
your main timeline or stay in a separate partner view.

Chosen over shared albums because it needs **zero maintenance**: an external
library is not an album, and newly indexed photos never auto-join an album, so
album membership would need topping up forever.

> **Partner sharing is all-or-nothing.** It shares a person's *entire* library —
> there is no way to hold some photos back. This deliberately overrides the
> per-person privacy that the Nextcloud mounts enforce: Anette's `Photos` folder
> stays private in Nextcloud while her photos are visible to the family in
> Immich. Fine for a household that wants it, but everyone should know that is
> the arrangement.

## Browsing by folder

Immich can browse the real directory structure, so organising `_Shared` into
subfolders (by year, event, trip) shows up directly in Immich with no album
admin. Enable per user at `/user-settings` → **Features** → **Folders**; it is
off by default and each person must enable it for themselves. Stored as
`preferences.folders = {"enabled": true, "sidebarWeb": true}` — `sidebarWeb`
puts it in the left sidebar.

**Immich does not create albums from folder names**, and there is no setting to
make Folders the landing page — the home page is always the chronological
timeline. Use folders for organisation and albums only for curated sets that
need sharing outside the family.

Check libraries do not overlap (a parent-folder library would silently pull in
everyone else's photos):

```bash
PGPOD=$(kubectl -n nextcloud get pod -l app=immich-postgres -o jsonpath='{.items[0].metadata.name}')
kubectl -n nextcloud exec $PGPOD -- psql -U immich -d immich -c "
  SELECT u.name AS owner, l.\"importPaths\" FROM library l
  JOIN \"user\" u ON u.id=l.\"ownerId\" ORDER BY u.name;"
```

## Faces

- **Face detection** finds faces; **facial recognition** groups them into people.
  A wrong grouping only needs *Facial Recognition → RESET* on
  `/admin/jobs-status`, not the much heavier Face Detection reset.
- **Merging people is irreversible** — Immich has no unmerge. If two clusters
  *might* be the same person, leave them; you can always merge later.
- **Name people only after jobs finish.** Clusters keep consolidating while
  recognition runs, and naming mid-run is what invites bad merges.
- If distinct people keep getting merged automatically, lower *Maximum
  recognition distance* (Settings → Machine Learning) rather than resetting
  repeatedly — a reset re-runs the same algorithm on the same data and reaches
  the same answer.

## Gotchas

- **Immich's mount is read-only.** Anything that writes to the volume (imports,
  moving files between folders) must go through Nextcloud.
- **New files need a scan.** Photos uploaded *through* Nextcloud appear there
  instantly; only the Immich index waits. Immich rescans external libraries on
  the cron in Settings → External Library → Periodic Scanning (set to hourly,
  `0 * * * *`; the shipped default is nightly). Files written directly to the
  volume, bypassing Nextcloud, also need
  `occ files:scan --path="<uid>/files/Photos"` for Nextcloud to notice them.
- **Do not enable Library Watching to get faster updates.** Beyond being marked
  experimental, it relies on filesystem events, and this volume is NFS (Longhorn's
  RWX share-manager). inotify does not reliably report writes made by *other* NFS
  clients, and Nextcloud writes from a different mount than Immich reads from — so
  the watcher would likely never fire for exactly the uploads that matter. Use a
  tighter cron instead; a scan with nothing to do is cheap.
- **Bulk imports are memory-hungry.** Thumbnails, transcoding, face detection and
  CLIP embedding all run concurrently; `immich-server` is set to 12Gi for this
  reason. If it is OOM-killed mid-import, either raise the limit further or lower
  the per-job concurrency in Settings → Job Settings.
- **Stalled jobs.** If a job sits `Active` with no progress and no log output,
  the worker died mid-job (BullMQ marks it active with no owner) — typically
  after an OOM. Restarting `immich-server` lets BullMQ requeue it. Confirm by
  checking whether Postgres is actually idle:
  `SELECT COUNT(*) FROM pg_stat_activity WHERE datname='immich' AND state<>'idle';`
- **The immich-charts Helm chart lags upstream releases** (0.12.0 still pinned
  v2.6.3 when Immich was on v3.1.0), so the image tag is overridden in
  `apps/immich/values.yaml`. Check Immich's release notes for breaking changes
  before bumping, especially once libraries hold real photos.
- **Public link shares bypass all of this.** A link-shared file is reachable by
  anyone with the URL regardless of family grouping. Audit with:

  ```bash
  kubectl -n nextcloud exec nextcloud-mariadb-0 -c mariadb -- mysql -u nextcloud -p"$DBPW" nextcloud -N -e \
    "SELECT uid_owner, share_with, file_target FROM oc_share WHERE share_with IS NULL;"
  ```
