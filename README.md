# public-vault

A public [Obsidian](https://obsidian.md) vault for the hands-on layer of the
work — lab and CTF writeups, disclosed-CVE reproductions, and the atomic notes
taken while doing them. This is the primary-source material; the finished
essays and the condensed cheat sheets live on the site.

公開 Obsidian vault。実作業レイヤ (ラボ/CTF の writeup・公開 CVE の再現・その過程で
とった atomic note) を置く一次資料。完成した考察エッセイと凝縮チートシートはサイト
側にあり、本 vault はその手前の生の記録を担う。

## Where things live

| Layer | Where | What |
|---|---|---|
| Essays | [blog](https://d0me-d0me.github.io) `_posts` | Finished offense/defense concept pieces |
| Cheat sheets | [refs](https://d0me-d0me.github.io) | Condensed operational references |
| **Hands-on notes** | **this vault** | **Writeups, CVE reproductions, atomic notes** |

Finished conceptual writing belongs on the blog, not here — this vault stays
upstream of it.

## Structure

| Folder | Contents |
|---|---|
| [`writeups/`](writeups/index.md) | CTF and lab machine writeups |
| [`cves/`](cves/index.md) | Reproductions of disclosed CVEs |
| [`notes/`](notes/index.md) | Atomic field notes that feed the cheat sheets |
| [`templates/`](templates/) | Note templates |

Start from the vault home: [`index.md`](index.md).

## Conventions

- Every technique note carries a **Detection & mitigation** section — the
  defensive counterpart is part of the note, not an afterthought.
- Lab and CTF writeups cover only retired or explicitly writeup-permitted
  targets.
- Links are plain relative Markdown so notes resolve both in Obsidian and on
  GitHub.

## Using it in Obsidian

Open this repository as a vault (Obsidian → Open folder as vault). The
`.obsidian/` workspace directory is git-ignored, so local UI state is not
published.
