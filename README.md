# Palmetto

**A clean-room runtime that runs an Xbox Series GTA VI build natively on Windows, built on your own PC from your own copy of the game.**

Palmetto is the local continuation of **[ItsNotRexed/Leonida](https://github.com/ItsNotRexed/Leonida)**. That project's
upstream repository went private, so this fork carries the work forward in the open: the same runtime, the same
clean-room approach, plus the boot fixes measured on this machine.

> [!IMPORTANT]
> This repository contains **no game, no keys and no executable**, and it does not explain how to obtain them. You
> need your own copy, with its files already readable on disk. Nothing here decrypts or works around any protection.
> Everything a build creates — translated shaders, extracted assets, the runtime folder, your saves — comes from your
> game and stays on your PC. Never share those files.

## Status

The runtime maps the title's PE image, resolves its imports through shims, reimplements the console's D3D12 layer on
Vulkan, and translates the console's shader corpus to SPIR-V. All of that works. The boot is not yet a frame.

| Piece | State |
|---|---|
| Shader corpus | **26,498 / 26,503** modules translated and `spirv-val` clean (2 shards crashed under load, repaired solo, merged) |
| PE loader, TLS, heap, file system | working |
| Xbox D3D12 → Vulkan device | working on an RTX 4090 (subgroup 32, bindless, Resizable BAR) |
| Console DirectStorage | working (`DStorageGetFactory`, `IDStorageFactoryX::OpenFile`) |
| **Retail boot** | **2,238 D3D12 calls**, then a stop on a compute pipeline the translator cannot build yet |

Every boot stop is documented with a measurement rather than a claim; the full ladder this fork started from is in
[docs/retail_boot_chain.md](docs/retail_boot_chain.md), and all thirteen of its rows are cleared here.

### The current frontier

The run now reaches root-signature and compute-pipeline creation, then stops on the title's own null-pipeline call: every
`DeserializeComputePipelineStateX` fails because the FidelityFX programs it names are among the 29 that do not all
translate yet. 19 of them do. The ten that do not split into two measured classes — a scalar-register file wider than
LLVM's model (which `tools/widen_smem_decode.py` repairs, and which recovered one program) and inline-literal encodings
the disassembler prints canonically (open). [docs/boot_frontier.md](docs/boot_frontier.md) has the measurements and the
method that produced them.

## Build

Requirements: **your own copy** of the Xbox Series package (1.0.2.1), Windows 10/11 x64, ~400 GiB free for the
full build, Python 3.12+, .NET 8 SDK, Visual Studio 2022 with the C++ tools and Windows SDK 10.0.26100+,
[Vulkan SDK](https://vulkan.lunarg.com/) (`spirv-as`, `spirv-val`), LLVM 21+ with the **AMDGPU** target for
`llvm-mc` (the official Windows build has none; `scripts\build_llvm_mc.ps1` builds one), and `ffmpeg` on PATH.

def test_restore_vop3p_dot4_recovers_raw_fields(self):
    raw = bytes.fromhex("08 50 16 CC 01 05 0E 0C")
    enc = bytes.fromhex("08 40 16 CC 01 05 0E 1C")

    ins = {"op": "v_dot4_i32_i8", "mods": [], "enc": ""}

    self.assertTrue(
        rdna2spirv.restore_vop3p_dot4(ins, raw, enc)
    )
    self.assertIn("op_sel:[0,1,0]", ins["mods"])
    self.assertIn("op_sel_hi:[1,0,1]", ins["mods"])
    self.assertEqual(ins["enc"], raw.hex())


def test_restore_vop3p_dot4_rejects_foreign_bit_flip(self):
    raw = bytes.fromhex("08 50 16 CC 01 05 0E 0C")
    enc = bytes.fromhex("08 40 16 CC 01 05 0E 1C")

    corrupted = bytes([raw[0] ^ 0x01]) + raw[1:]

    ins = {"op": "v_dot4_i32_i8", "mods": [], "enc": ""}

    self.assertFalse(
        rdna2spirv.restore_vop3p_dot4(ins, corrupted, enc)
    )


def test_restore_vop3p_dot4_rejects_wrong_opcode(self):
    raw = bytearray.fromhex("08 50 16 CC 01 05 0E 0C")
    enc = bytes.fromhex("08 40 16 CC 01 05 0E 1C")

    raw[2] = 0x17

    ins = {"op": "v_dot4_i32_i8", "mods": [], "enc": ""}

    self.assertFalse(
        rdna2spirv.restore_vop3p_dot4(ins, bytes(raw), enc)
    )

```powershell
git clone --recursive https://github.com/luizwidmer/palmetto
cd palmetto
python -I make_game.py --find                 # locate your copy
python -I make_game.py --check                # verify it
python -I make_game.py --game "<pkg>" --out "<out>" --work "<work>" --stages shaders,runtime
```

`scripts\build_runtime.ps1` builds `gta6rt.exe` on its own. Then:

```powershell
gta6rt.exe --dry-run <pkg>\game_scarlett_master.exe --game-dir <pkg>   # map the image, list imports
gta6rt.exe --run     <pkg>\game_scarlett_master.exe --game-dir <pkg> --overlay <dir> --trace-d3d12 --trace out.jsonl
```

For a comparable one-line summary of any attempt, use the harness:

```powershell
python -I tools/boot_report.py --exe runtime\build\gta6rt.exe `
    --image <pkg>\game_scarlett_master.exe --game-dir <pkg> --overlay <dir> --timeout 150
```

## How it works

| Piece | What it does | Where |
|---|---|---|
| Shaders | carve DXBC, unpack native RDNA2, disassemble with LLVM `gfx1013`, translate to SPIR-V | `tools/shader_carve.py`, `fssm_unpack.py`, `shader_disasm.py`, `rdna2spirv.py` |
| Runtime | maps the PE image, resolves imports to shims, Xbox-ABI proxies over a Vulkan-backed D3D12 device | `runtime/` |
| Assets | validated resources inside RPF9 archives, texture pools, audio banks | `tools/rpf9dump`, `tools/ttpf_assets.py` |
| Progress | one offline page of what passes and what blocks next | `tools/progress_dashboard.py` |

Format and contract notes are indexed in [docs/README.md](docs/README.md).

## Rules

- **No game data, ever** — no archives, executable, carved files, shaders in any form, textures, models, audio, or
  logs and reports that quote them. Share outcomes (counts, pass/fail, error classes), not inputs.
- **No keys, decryption or protection workarounds.** The tools only read files that are already plain on disk.
- Short excerpts needed to explain a format (a few header bytes, one instruction) are fine in docs and comments.

## Credits

The design, the runtime, the shader translator and the documentation are the work of the **Leonida** authors and its
contributors — see [ItsNotRexed/Leonida](https://github.com/ItsNotRexed/Leonida). This fork continues that work and
adds the boot measurements above; the original authorship is theirs, not ours.

Also: [rpf9-explorer](https://github.com/breadbbbb/rpf9-explorer) (RPF9 decoders), [Mesa](https://gitlab.freedesktop.org/mesa/mesa)
(descriptor layout, ACO/RADV reference), [libHttpClient](https://github.com/microsoft/libHttpClient) (XTaskQueue),
[LLVM](https://llvm.org/) (AMDGPU disassembler), [SPIRV-Tools](https://github.com/KhronosGroup/SPIRV-Tools).

Grand Theft Auto and all related names are trademarks of their owners. This project is not affiliated with or
endorsed by them, nor with Microsoft or the original Leonida project.

## License

GPLv3 or later. See [LICENSE](LICENSE). Third-party code keeps its own license.
