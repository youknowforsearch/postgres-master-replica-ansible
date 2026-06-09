# Offline Docker bundle (split for GitHub)

The full archive is ~110 MB (over GitHub's 100 MB file limit), so it is stored as **two parts** that ship with this repo:

```text
files/docker-deb.tgz.part00   (~55 MB)
files/docker-deb.tgz.part01   (~55 MB)
```

The `docker_airgapped` role copies both parts to each host, reassembles them, and runs `install.sh`.

The monolithic archive is gitignored:

```text
files/docker-deb.tgz            # local only — do not commit
```

## Regenerate parts after updating the bundle

From the repo root:

```bash
cd files
split -n 2 -d -a 2 docker-deb.tgz docker-deb.tgz.part
ls -lh docker-deb.tgz.part*
```

Verify reassembly:

```bash
/bin/cat docker-deb.tgz.part00 docker-deb.tgz.part01 > /tmp/docker-deb-test.tgz
cmp -s docker-deb.tgz /tmp/docker-deb-test.tgz && echo OK
rm /tmp/docker-deb-test.tgz
```

## Bundle layout

The tarball must extract to a directory containing `install.sh` and `.deb` packages:

```text
docker-deb/
├── install.sh
└── *.deb
```

Adjust `docker_airgapped_extracted_dir` in `roles/docker_airgapped/defaults/main.yml` if your layout differs.

## Skip when Docker is already installed

```yaml
# group_vars/all.yml
docker_airgapped_enabled: false
```
