# Family photos: Nextcloud + Immich on one shared volume

Nextcloud and Immich both read the *same files on disk*. Nothing is synced or
duplicated between them: `family-photos-shared` (a Longhorn RWX PVC, see
`apps/family-photos/`) is mounted read-write into Nextcloud and **read-only**
into Immich, both at `/mnt/family-photos`.

Because Kubernetes PVCs are namespace-scoped — and Longhorn RWX volumes become
unstable when referenced by separate PV/PVC pairs across namespaces — Immich is
deployed into the `nextcloud` namespace rather than its own.

- **Nextcloud** is the drive: upload, drag-and-drop, bulk imports (e.g. rclone
  from OneDrive), sharing with people outside the family.
- **Immich** is the viewer: timeline, map, faces, semantic search. It indexes the
  same folders via its *External Libraries* feature and never owns the originals.

## Layout on the volume

```
/mnt/family-photos/
  Thomsen/
    Chris/          # one folder per person
    Thomsen/
    _Shared/        # whole-family photos
```

Isolation between families is enforced by *who each mount is applicable to*, not
by folder naming. A family only ever sees its own subtree.

**Immich is opt-in per family.** Only the Thomsen family uses it, so only
`Thomsen/` exists on this volume. The Roer family uses plain Nextcloud — their
photos stay in their normal Nextcloud home storage and never touch this volume.

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
| `Immich` | `Thomsen/Chris` | user `Chris` |
| `Family Photos` | `Thomsen/_Shared` | group `family-thomsen` |

Everyone in a family sees the same mount names; the folder each resolves to
differs per user.

> **Never name a mount after a folder the user already has.** Nextcloud mounts
> external storage *over* an existing folder of the same name, so a mount called
> `Photos` silently hides a real `Photos` folder — the files are untouched on
> disk but vanish from the UI, which looks exactly like data loss. Three of four
> users here already had a `Photos` folder. Check first:
>
> ```bash
> kubectl -n nextcloud exec $NCPOD -c nextcloud -- ls /var/www/html/data/<uid>/files/
> ```
>
> `Immich` was chosen as the mount name precisely because it collides with
> nothing and states what the folder feeds.

> External Storage was chosen over Groupfolders deliberately: Groupfolders keeps
> data inside Nextcloud's own opaque internal directory structure, which Immich
> cannot read directly. Local external storage points straight at the real path,
> so both apps see identical human-readable folders.

## Adding a new person

All commands run inside the Nextcloud pod:

```bash
NCPOD=$(kubectl -n nextcloud get pod -l app.kubernetes.io/name=nextcloud -o jsonpath='{.items[0].metadata.name}')
alias occ="kubectl -n nextcloud exec $NCPOD -c nextcloud -- php occ"
```

1. **Create the account** (or via the admin UI):

   ```bash
   occ user:add --display-name "Their Name" theiruid
   ```

2. **Add them to their family group** — this alone grants access to that
   family's `_Shared` folder:

   ```bash
   occ group:adduser family-thomsen theiruid    # or family-roer
   ```

3. **Create their personal folder**, owned by `www-data` (uid 33) so the
   Nextcloud web process can write to it. Immich runs as root and reads
   regardless of ownership:

   ```bash
   kubectl -n nextcloud exec $NCPOD -c nextcloud -- sh -c '
     mkdir -p /mnt/family-photos/Thomsen/TheirName &&
     chown 33:33 /mnt/family-photos/Thomsen/TheirName &&
     chmod 755 /mnt/family-photos/Thomsen/TheirName'
   ```

4. **Create the external storage mount and scope it to just them.** The
   `files_external:applicable` step is not optional — a freshly created system
   mount is applicable to *every* user until you restrict it:

   ```bash
   # Check the name is free first - see the shadowing warning above
   occ files_external:create "Immich" local null::null \
     -c datadir=/mnt/family-photos/Thomsen/TheirName
   # note the returned id, then:
   occ files_external:applicable <id> --add-user theiruid
   occ files_external:verify <id>          # expect: status: ok
   ```

   If the person already has photos in their Nextcloud home that should appear
   in Immich, they must be *copied onto the shared volume* — Immich cannot read
   Nextcloud's internal storage. Copy first, verify with checksums, and only
   delete the originals once both Nextcloud and Immich show the photos:

   ```bash
   SRC=/var/www/html/data/theiruid/files/Photos
   DST=/mnt/family-photos/Thomsen/TheirName
   kubectl -n nextcloud exec $NCPOD -c nextcloud -- sh -c "
     cd $SRC && find . -type f -print0 | xargs -0 md5sum | sort -k2 > /tmp/src.txt
     rsync -a $SRC/ $DST/ && chown -R 33:33 $DST
     cd $DST && find . -type f -print0 | xargs -0 md5sum | sort -k2 > /tmp/dst.txt
     diff -q /tmp/src.txt /tmp/dst.txt && echo VERIFIED_IDENTICAL"
   ```

5. **Verify the scoping before handing over the account.** This lists every
   mount with who can reach it — check no mount has both an empty user list and
   an empty group list, which would mean it is globally visible:

   ```bash
   occ files_external:list --output=json | jq -r \
     '.[] | "id=\(.mount_id) \(.mount_point) -> \(.configuration.datadir)\n   users=\(.applicable_users) groups=\(.applicable_groups)"'
   ```

6. **Add them to Immich** (see below) if they want the photo-browsing UI.

To confirm the mounts actually resolve to different backing folders per user
(the real isolation test, rather than trusting the config):

```bash
DBPW=$(kubectl -n nextcloud get secret nextcloud-db-credentials -o jsonpath='{.data.mariadb-password}' | base64 -d)
kubectl -n nextcloud exec nextcloud-mariadb-0 -c mariadb -- mysql -u nextcloud -p"$DBPW" nextcloud -N -e "
  SELECT m.user_id, m.mount_point, s.id FROM oc_mounts m
  JOIN oc_storages s ON s.numeric_id=m.storage_id
  WHERE m.mount_point LIKE '%Photos%' ORDER BY m.user_id;"
```

Mounts only appear in `oc_mounts` after the user's storage is first accessed, so
a new person's rows show up after their first login (or after
`occ files:scan <uid>`).

## Adding a person to Immich

Immich accounts are separate from Nextcloud accounts — there is no shared login.

1. Create the account: `https://photos.hivemindcloud.dk/admin/users`
   (Avatar → Administration → Users → Create user).
2. Create the library: `https://photos.hivemindcloud.dk/admin/library-management`
   (Avatar → Administration → **External Libraries**). Note this is a different
   page from *Settings → External Library*, which only holds the global watch
   and scan-schedule options — there is no "create" button there.
   - **Create Library**, and pick the owner. The owner cannot be changed later.
   - Under **Folders** → **Add**, enter `/mnt/family-photos/<Family>/<Person>`.
     Use the path as the *container* sees it, which is what is written above —
     not a host path.
   - Click **Scan**.
3. Immich indexes in place and never writes to the volume — its mount is
   read-only, so originals cannot be modified or deleted by Immich.

Immich has no concept of "families". Isolation there comes from each library
being owned by one user, plus only sharing albums within a family.

For the `_Shared` folders, either give one person a library pointing at
`_Shared` and share the resulting album with the rest of that family, or give
each family member their own library on the same path.

## Gotchas

- **Immich's mount is read-only.** Anything that needs to *write* to the volume
  (bulk imports, moving files between folders) must go through Nextcloud.
- **New files need a scan to appear.** Nextcloud picks up out-of-band changes via
  `occ files:scan --path="<uid>/files/Photos"`; Immich rescans external libraries
  on its own schedule or on demand from the admin UI.
- **The immich-charts Helm chart lags upstream releases** (0.12.0 still pinned
  v2.6.3 when Immich was on v3.1.0), so the image tag is overridden in
  `apps/immich/values.yaml`. Check Immich's release notes for breaking changes
  before bumping it, especially once the library holds real photos.
- **Public link shares bypass all of this.** A file shared by link is reachable
  by anyone with the URL, regardless of family grouping. Audit with:

  ```bash
  kubectl -n nextcloud exec nextcloud-mariadb-0 -c mariadb -- mysql -u nextcloud -p"$DBPW" nextcloud -N -e \
    "SELECT uid_owner, share_with, file_target FROM oc_share WHERE share_with IS NULL;"
  ```
