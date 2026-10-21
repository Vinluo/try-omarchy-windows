# Intel Arc B390: mutable buffer upload stalls

Recorded on physical Windows hardware on 2026-10-21. This is evidence for a
renderer compatibility patch, not acceptance of a new public runtime release.

## Environment

| Component | Tested configuration |
| --- | --- |
| Host | Windows 11 Home, build 26200.9711, 32 GiB RAM |
| CPU | Intel Core Ultra X7 358H, 16 logical processors |
| GPU | Intel Arc B390, driver 32.0.101.9150 |
| Additional display driver | GameViewer Virtual Display Adapter 15.6.5.199 |
| Launcher | Try Omarchy v0.12.0, updates disabled during testing |
| QEMU | Bundled 11.0.0, WHPX, host CPU model |
| Guest | Linux 7.2.9-arch1-1, Hyprland 0.56.2 |
| Resources | 8 vCPUs, 6144 MiB guest RAM, 4 GiB GPU hostmem |
| Display | 3120 x 1939 at approximately 120 Hz, scale 1 |
| Graphics | `virtio-vga-gl`, blobs and Venus enabled, SDL OpenGL |

Testing used independent copies of a cleanly stopped guest. The personal guest
and the preserved baseline disk were not used as writable test disks.

## Failure evidence

Opening the desktop menu, the Super+K keybinding viewer, and configuration
entries reproduced a Windows "not responding" window. Windows also recorded
QEMU application hangs. The launcher remained responsive, but QMP and guest
SSH stopped answering. Approximately 10 GiB of host RAM was still available;
no Windows display-reset or resource-exhaustion event was found in that period.

Two captured main-thread waits established both affected buffer classes:

| Upload | Storage | Transfer | Map flags |
| --- | --- | --- | --- |
| Index buffer, `GL_ELEMENT_ARRAY_BUFFER` | Mutable | Offset 0, 12 bytes | `WRITE`, `INVALIDATE_RANGE`, `UNSYNCHRONIZED` |
| Vertex buffer, `GL_ARRAY_BUFFER` | Mutable | Offset 0, 192 bytes | Same |

The second capture was obtained with an index-only experimental bypass, which
left the vertex-buffer mapping path unchanged. This is why the final patch
covers mutable buffer uploads rather than only element buffers.

The repeated stack was:

```text
NtGdiDdDDIWaitForSynchronizationObjectFromCpu
  Intel igvk64.dll / igxe3icd64.dll
  glMapBufferRange
  vrend_renderer_transfer_write_iov
  vrend_decode_transfer3d
  virtio_gpu_virgl_process_cmd
  qemu_main_loop
```

An additional live sample several minutes later remained in the same driver
wait. This locates the blocking API boundary; it does not establish the
closed-source driver's internal dependency cycle.

## Candidate validation

The final candidate keeps GPU rendering and uses the existing iovec
`glBufferSubData` callback for mutable buffers on Intel Windows. Immutable
storage retains its mapping path.

- The new buffer-upload regression failed against the unmodified renderer.
- It passes with the patch, compiled on Windows/MSYS2 and Linux. It checks
  Windows/non-Windows routing, immutable storage, other/null vendors,
  synchronized/unsynchronized maps, partial multi-iovec data and error fallback.
- Existing handle, host-memory and image-capability renderer checks pass.
- Linux release-helper tests: 27 passed; GPU-probe tests: 15 passed.
- The source lock and patch digests validate.
- A live breakpoint confirmed a 192-byte upload through `iov_buffer_upload`
  and `vrend_read_from_iovec_cb` in the candidate, rather than the stalled map.
- Fifty iterations opening/closing both the root menu and keybinding viewer
  completed from 12:19:38 to 12:20:59 UTC+08. QEMU and guest control remained
  responsive. Operator interaction is additional coverage, not a timed soak.
- The operator also repeated Config -> Wayland and Super+K on the final
  candidate and reported that these previously failing operations worked.

Candidate renderer SHA-256:
`9cd95e2c091edfeb144bb8d86d0a1456ac74eb43ec12ea52acee7793da096d48`.
It was built from the locked renderer source plus the recipe patches with
MSYS2 UCRT64 GCC 16.2.0, release optimization and debug symbols.

## Limits

Only this Intel GPU/driver combination has physical coverage. A one-hour soak,
upload-throughput comparison, other drivers and complete runtime packaging are
not claimed. The public guest/runtime pin is unchanged. Local hang dumps are
retained privately; guest disks, credentials and full memory dumps are not part
of this evidence.
