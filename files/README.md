# Offline Docker bundle (Stored on Git LFS)

The air-gapped Docker archive is stored as a single file tracked by **Git LFS**:

```text
files/docker-deb.tgz   (~110 MB)
```

GitHub's normal 100 MB file limit does not apply — LFS stores the binary separately.

## Clone / checkout

```bash
# one-time on your machine
git lfs install

# clone (LFS files download automatically when LFS is installed)
git clone <repo-url>
cd postgres-master-replica-ansible

# existing clone — fetch the bundle if you only have a pointer file
git lfs pull
```

Verify the real archive is present (not a tiny LFS pointer):

```bash
ls -lh files/docker-deb.tgz    # should be ~110 MB
```

## Bundle layout

The tarball extracts to a directory containing `install.sh` and `.deb` packages:

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

## Update the bundle

Replace `files/docker-deb.tgz`, then:

```bash
git add files/docker-deb.tgz
git commit -m "Update offline Docker bundle"
git push
```

Git LFS uploads the new object automatically when LFS is configured on the remote (GitHub enables LFS by default).
