# public-vault

A public [Obsidian](https://obsidian.md) vault of long-form security notes —
lab writeups, disclosed-CVE reproductions, and field notes from both sides of
security. The terminal-style cheat sheets live on the site; this vault is the
longer working notes behind them.

公開 Obsidian vault。ラボの writeup・公開 CVE の再現・攻撃/防御両面のフィールド
ノートを長文で記録する。要点を凝縮したチートシートはサイト側にあり、本 vault は
その背後の作業ノートを置く場所。

- Site / cheat sheets → https://d0me-d0me.github.io
- Profile → https://github.com/d0me-d0me

## Structure

| Folder | Contents |
|---|---|
| [`writeups/`](writeups/index.md) | CTF and lab machine writeups |
| [`cves/`](cves/index.md) | Reproductions of disclosed CVEs |
| [`notes/`](notes/index.md) | Atomic field notes |
| [`concepts/`](concepts/index.md) | Longer concept notes bridging offense and defense |
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
