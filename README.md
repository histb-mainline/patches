# histb patch store (quilt)

Out-of-tree patch series for the HiSilicon STB platforms (Hi3798MV100 /
Hi3716CV200 / Hi3798MV200 / Hi3798MV200H), laid out the way OpenWrt lays out
`target/linux/*/{backport,pending,hack}-<kver>`: the patch files and the
`series` file here are the source of truth, a git branch is not. Kernel trees
are disposable and are extracted from the baseline commit named in `baseline`.

This file states only what is specific to this store - layout, class order,
numbering, size bands and the grouping decisions. quilt itself is documented by
quilt. Nothing here checks your work: break a rule below and the next `push` is
what tells you. Cutting a submission out of the series, writing its introduction
and sending it are a separate upstreaming process, not part of porting.

## Layout

```
series              the order patches apply in (bucket order, file-name order inside a bucket)
baseline            kernel version + commit every patch applies to
quilt.conf          --fuzz=0, -p ab refresh, and QUILT_PATCHES pointing here
backport-7.4/       patches taken from another kernel version (none yet)
pending-7.4/        submittable patches
hack-7.4/           local-only patches, not part of any submission
```

`<bucket>-7.4` is named from `KERNEL_VERSION` in `baseline`; how many patches and
submissions there are right now is a question for `series`, not for this file.

## What this directory expects a tree to do

`quilt.conf` defaults `QUILT_PATCHES` to the directory it lives in, so a tree
only has to name the file:

```sh
quilt --quiltrc=<this directory>/quilt.conf push -a
```

`new` and `refresh` then write into this directory, `series` included - there is
no staging copy and nothing to sync back. `.pc/` stays in the tree, so an arm
and an arm64 tree can work against one store. Patches apply with `--fuzz=0`.

Get a tree by extracting `BASE_SHA` from `baseline`
(`git -C <kernel repository> archive <BASE_SHA> | tar -x -C <tree>`, about
1.9 GB) and take the kernel version and configuration from `baseline` too.

Renumbering or splitting patches is a file rename plus a `series` edit; to
rebuild `series` after bulk changes:

```sh
for b in backport pending hack; do
	find $b-7.4 -maxdepth 1 -name '*.patch' 2>/dev/null | sort
done >series
```

Do not run `quilt init` in this directory: `series` is committed here, and init
would add a `.pc/`.

## Two states of the series

| state | what |
|---|---|
| submittable | everything up to the last `series` entry before the first `hack-` one |
| local | the whole series, bring-up instrumentation included |

The local-only content is the `hack-7.4/` bucket, numbered 990-SS so it can only
ever sit at the end; 990 is not a submission number.

## Naming

`NNN[-SS]-<chip>-<shortname>.patch`; backport patches carry an extra
`-vX.Y-` segment for the kernel version they came from. `<chip>` is `mv100`,
`cv200`, `mv200`, `mv200h` or `all` (`mv200` is the Hi3798MV200, a different
chip from the Hi3798MV200H `mv200h`).

**`NNN` is one submission** - the series that goes to one maintainer - and `SS` is
the position of a patch inside it. A submission uses `NNN-SS` throughout, never
bare `NNN` alongside `NNN-SS`.

The hundreds digit of `NNN` is the submission class, and class order is
dependency order (a provider always precedes its consumers):

| range | class | ordered within the class by |
|---|---|---|
| 0xx | shared ABI prerequisites: dt-bindings consumed by 2+ submissions, vendor prefixes, txt-to-YAML conversions | being consumed earliest |
| 1xx | fixes to existing drivers and cores, arch/machine/Kbuild changes | size, ascending |
| 2xx-8xx | new driver introductions (a driver's own binding stays in the same `NNN`) | size, ascending - the hundreds digit *is* the size band |
| 9xx | board integration: one `NNN` per board, `SS` order is SoC dtsi, board dts, enabled nodes | not by size |
| 990 in `hack-7.4/` | local-only content: bring-up instrumentation, bench harness | branch order |

Size bands, counted as added lines across the whole submission:

| band | limit | what lands there |
|---|---|---|
| 2xx | 250 | watchdog, thermal, regulator, keymaps, net-phy, phy |
| 3xx | 800 | `arch/arm/mach-hisi/` (675) |
| 4xx | 2,000 | `drivers/pinctrl/hisilicon/` (1,012), `sound/soc/hisilicon/` (1,620) |
| 5xx | 5,000 | `drivers/clk/hisilicon/` (3,307), `drivers/gpu/drm/bridge/` (2,430) |
| 6xx | 10,000 | `drivers/gpu/drm/histb/` (7,690), `drivers/media/platform/hisilicon/` (6,950) |
| 7xx | 20,000 | - |
| 8xx | unlimited | `.../histb-vdec/` (24,160) |

## Grouping rules

1. Fix or new driver: does the patch create the driver source file (a `--- /dev/null`
   hunk for a `.c`/`.h`)? If yes, it is an introduction.
2. Changes to a driver this series itself introduces are folded into the tail of
   that submission's `SS` range - they are not independently submittable.
3. A dt-bindings or shared id header consumed by 2 or more submissions belongs in
   0xx. Inside a single submission it cannot be cherry-picked alone, and it would
   not survive a per-patch build test.
4. **If several patches touch the same file, they belong to the same submission.**
   In a fresh series, the context of the later patch does not match the result of
   the earlier one once they are split, and `--fuzz=0` refuses outright. Same-file
   overlap is what decides whether id headers, shared `Kconfig`/`Makefile` edits
   and `MAINTAINERS` entries stay with their provider's submission.

## Submission headers

- The `Subject:` in each patch file is a **bare** subject with no `[PATCH...]`
  token, and a `quilt new` patch has no `Subject:` until you add one with
  `quilt header`.
- Per-patch authorship and date ride on `Replace-From:` / `Replace-Date:`
  headers, which the upstreaming process replays patch by patch.

Splitting a submission out of `series` and writing what goes on top of it is not
this repository's business: the porting workflow stops at the patch files, and
upstreaming decides the rest.
