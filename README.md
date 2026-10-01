# pcie-link-map

**Your PCIe tool is lying to you, and this shows you the truth.**

No root. No dependencies. One file.

```
$ ./pcie-link-map --gpu
address       capable     max GB/s  live now      note
0000:03:00.0  x16 Gen5    64.00     Gen5 x16      full width
0000:09:00.0  x4 Gen3      4.00     Gen3 x4       card says x16, host gives x4
0000:11:00.0  x1 Gen3      1.00     Gen3 x1       card says x16, host gives x1
```

---

## The problem

You want to know how many PCIe lanes your GPU actually has. You check, and you get **x16**.

It's wrong. Here's what every tool reports:

| tool | says | what it actually measured |
|---|---|---|
| `lspci -vv` | `LnkSta: Width x16` | the link to **the bridge chip on the card** |
| `sysfs` | `current_link_width` = 16 | the same bridge link |
| amdgpu | `pp_dpm_pcie` → `x16` | the same bridge link |
| GPU-Z / HWiNFO | often the adapter's max | the same bridge link |

**None of them measure the link to your CPU.** A card can report x16 in all four while its real host link is **one lane** — 16x less bandwidth than you think.

This bites eGPU and OCuLink dock users constantly, because a dock's whole job is to give you fewer lanes. People spend days reseating cards, swapping cables and hunting BIOS settings, because the tool says x16 and the machine feels slow.

### Real reports of exactly this

- *"`LnkSta: Speed 2.5GT/s (downgraded), Width x4`"* — reading the wrong field, on an OCuLink dock
- *"`lspci`'s `LnkSta` field is dynamic when power save mode is active"* — discovered by hand
- *"PCIe v5.0 x16 (32.0 GT/s) @ x4 (2.5 GT/s)"* — the card's capability shown next to a live reading, and read as one number

### The second trap: a link is not a number

PCIe **downclocks to Gen1 at idle**. So:

- a single reading is a photograph of a moment, not a property of your machine;
- two people comparing "their" link width may be reading different states of the same box;
- a link that *never* comes up under load is a real fault — and it looks identical to power saving if you only sample once.

`pcie-link-map` reports capability and live state **separately**, and can watch the link to tell the two apart.

## What it does

- **CAPABILITY** — the narrowest max width/speed along the whole path from the root complex down. Stable, and what you size against.
- **LIVE** — the narrowest current width/speed. Drops at idle; tagged when it does.
- **`--watch N`** — samples the link. If it never comes up, tells you it's a fault, not power saving.
- **`--diagnose`** — a plain verdict: *"LINK LIMITED: the card reports x16, the host provides x1. This is the slot/dock/riser, not the card."*

## The rule that makes it work

A device's link is **the minimum negotiated width along the whole chain** from the root complex:

```
/sys/devices/pci0000:00/0000:00:03.4/0000:0f:00.0/0000:10:00.0/0000:11:00.0
  ^root complex      ^x1              ^x16         ^x16         ^x16
                     ^^^ THIS is what binds
```

The GPU's own entry says x16. The root port says x1. **The minimum is the truth.**


## For machines and agents

A table is for a person. **`--json` is for anything that has to consume it** — one document,
flat schema, stable keys:

```json
{"tool": "pcie-link-map", "version": 1,
 "devices": {"0000:11:00.0": {
    "capable":      {"width": 1,  "gen": "Gen3", "gbps": 1.0},
    "live":         {"width": 1,  "gen": "Gen3"},
    "card_reports": {"width": 16},
    "binding": "0000:00:03.4",
    "link_limited": true,
    "downclocked": false,
    "is_gpu": true,
    "verdict": "LINK LIMITED: the card reports x16, the host provides x1..."
 }}}
```

The key that makes it useful: **`link_limited` is a boolean.** True means the host gives fewer
lanes than the card advertises — the exact condition this tool exists to detect, as a value a
program can branch on rather than prose it has to parse.

`--json --watch 10` adds an `observed` array per device, so a caller can see whether a link
ever comes up or is stuck low.

## Install

```bash
curl -Lo pcie-link-map \
  https://raw.githubusercontent.com/Sophia-Thickums/pcie-link-map/main/pcie-link-map
chmod +x pcie-link-map
./pcie-link-map --gpu
```

No build step, no dependencies — it is one Python 3 file. Or just clone it:

```bash
git clone https://github.com/Sophia-Thickums/pcie-link-map
```

Nothing to build, nothing to install, no root, no packages. Python 3 standard library only.

## Usage

```
pcie-link-map                 # devices whose link is limited (the interesting ones)
pcie-link-map --gpu           # display / 3D controllers
pcie-link-map --diagnose      # plain-language verdict per device
pcie-link-map --gpu --watch 20  # is the low link stuck, or just idle?
```

## One thing it can't tell you

**No riser can widen a slot.** If your board's slot is electrically x1, a riser gives you x1 on a longer cable — a riser *extends* a slot, it does not *convert* one. To check the physical slot:

```bash
sudo dmidecode -t slot
```

A slot whose `Type` reads `PCI Express 3 x1` is **x1 by design**, even if its name is `PCIEX16(G4)_3`. That is the slot working as specified, not a seating fault or a BIOS setting.

## Verified against

This tool's output was checked against the kernel's own account — the boot-time
`dmesg | grep "available PCIe bandwidth"` line — on a machine with four GPUs behind
three different kinds of link (a Gen5 x16 slot, an M.2→OCuLink Gen3 x4 dock, and a
physical Gen3 x1 slot). It reproduced the kernel's figures exactly, including naming
the same binding element.

**It is verified on one machine.** The minimum-width rule is what PCIe topology implies,
and it held there — but if this tool ever disagrees with your `dmesg`, **the kernel is
right and this tool is wrong.** Please open an issue with both outputs.

## License

MIT
