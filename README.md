<!-- SPDX-FileCopyrightText: 2026 roolrz -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# EROFS media reader

The `hyper_fs_erofs` library reads immutable EROFS images through a bounded random-read device.
The Native `fs-erofs` process adapts that interface to its exclusive block-volume
capability. Generic filesystem RPC, mount publication, and namespace symlink
resolution remain outside the format reader.

The supported format is uncompressed EROFS with 512, 1024, 2048, or 4096-byte
blocks, compact and extended inodes, flat contiguous data and inline tails,
regular files, directories, and symbolic links. A present superblock CRC32C is
validated. Unsupported incompatible features and inode layouts, including
compression, chunked files, external devices, and extended attributes, return
an explicit error. Device nodes, sockets, and FIFOs are also unsupported.

The parser uses safe Rust and checks disk offsets against the actual device
length. It caches one 4 KiB metadata block and one resolved path, without loading
whole files or images. Directory scans are limited to 16 MiB per directory;
paths and symlink targets to 4096 bytes; filenames to 255 bytes. HypeR's namespace
requires UTF-8 names. Symlink target bytes are returned without interpretation.
The device must remain immutable for the lifetime of a mounted reader.

`Volume::read_directory(path, cursor, output)` writes a directory batch of at
most 4 KiB using the common `hyper-filesystem` protocol. Each batch carries
names, complete supported attributes, per-entry continuation checkpoints and
an EOF flag. A zero cursor resolves the path; subsequent requests restore the
directory inode, block offset and entry position, without rescanning earlier
blocks. The reader validates the current block and its ordering against the
preceding block. Separate readers can interleave requests without shared scan
state. The indexed `entry(path, index)` interface has been removed.

Checkpoints are private to the kernel directory stream, bound to its mounted
directory; callers must not accept physical checkpoints from applications.
The parser checks cursor format, path fingerprint, inode geometry and directory
bounds, while the kernel owns authority and mount lifetime. The worker retains
no directory snapshot or per-reader allocation. Kernel directory/name/attribute
caches are optional and reclaimable under the same memory pressure as file
data pages; after eviction, the retained stream position still permits a
bounded read from the worker. The immutable EROFS device requires no media
mutation invalidation while mounted.

The implementation follows the published [EROFS core disk format](https://erofs.docs.kernel.org/en/latest/ondisk/core_ondisk.html).
It does not incorporate Linux filesystem implementation code.

Run the formal parser and worker module tests from the workspace root:

```sh
make test MODULE=fs-erofs
```

This selects the `hyper-fs-erofs-system` package in `svc/fs-erofs`, including its
`hyper_fs_erofs` parser library. It does not select the independent bootstrap copy.
The host [inspect example](examples/inspect.rs) lists a directory or emits file
and symlink bytes. From the workspace root, use the local SDK source patches that
host tests use (these crates are not published on crates.io):

```sh
CARGO_TARGET_DIR="$PWD/out/erofs-inspect" cargo run --locked \
  --manifest-path svc/fs-erofs/Cargo.toml -p hyper-fs-erofs-system \
  --config "patch.crates-io.hyper-abi.path='$PWD/sdk/abi'" \
  --config "patch.crates-io.hyper-os.path='$PWD/stdlib/rust/hyper-os'" \
  --config "patch.crates-io.hyper-rt.path='$PWD/stdlib/rust/hyper-rt'" \
  --config "patch.crates-io.hyper-sys.path='$PWD/stdlib/rust/hyper-sys'" \
  --config "patch.crates-io.hyper-filesystem.path='$PWD/stdlib/rust/hyper-filesystem'" \
  --example inspect -- svc/fs-erofs/tests/fixtures/core.erofs hello
```

The included fixture prints `EROFS reference fixture`. Replace the final image
and path arguments to inspect another uncompressed EROFS image; omit the path to
list its root directory. This is a host utility, not a Native executable.

## Workspace tests

Use the [HypeR workspace](https://github.com/HypeR-Proj/HypeR) to resolve dependencies:

```sh
python3 project.py test fs-erofs
```

Paths in the documentation are relative to the repo-managed workspace root.
Edit this repository and prepare the workspace again before building.
