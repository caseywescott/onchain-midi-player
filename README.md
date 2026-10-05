# onchain-midi-player

A Cairo class library for Starknet that gives an NFT's `token_uri` an onchain MIDI player. The player is an HTML page in the token's `animation_url`. The page holds the synth engine, the player, the token's MIDI, its sound settings and its SVG art, and it plays offline in the browser with no network requests. Your NFT contract supplies the MIDI, settings and art, and calls the class with `library_call` to get the pieces of its `token_uri`. The class is declared, never deployed. It has no storage, no constructor and no collection-specific logic. The engine is currently [TinySynth](https://github.com/Provable-Games/webaudio-tinysynth), a General MIDI synthesizer (oscillators and FM, no samples).

## How it works

Your contract holds the class hash and calls the class through `ITinySynthLibraryDispatcher`. Every function is a deterministic view, and the class touches none of your storage.

`token_uri` is nested base64 data URIs. The JSON is `data:application/json;base64,…`, and its `animation_url` is `data:text/html;base64,…`, the player page. The engine and player are the same for every token, so the class stores them already encoded and never encodes them at call time. Your contract encodes only per-token data, in pieces that are each a multiple of 3 bytes long, and joins them. That works because `b64(X ++ Y) == b64(X) ++ b64(Y)` when `len(X) % 3 == 0`.

```text
"data:application/json;base64,"
  ++ b64('{' members ',' <pad> '"image":"data:image/svg+xml;base64,')
  ++ b64(S)                        S = b64(svg) '"' <pad>, encoded once, used twice
  ++ b64(',' <pad>)
  ++ animation_url_segment()       ← class: the fixed page (engine and player), pre-encoded
  ++ midi_segment(midi, settings)  ← class: this token's settings and MIDI
  ++ b64(S)                        the art the page shows; closes both data URIs
  ++ b64('}')
```

`<pad>` is spaces between JSON tokens. They keep each piece a multiple of 3 bytes.

In the browser, the page shows the art at once. ▶ starts the MIDI, which loops at End-of-Track, and the art restarts in step with every pass. If the engine, the settings or the MIDI fails, ▶ stays disabled, the page shows the error, and the art stays. The page works in a sandboxed iframe and under a strict CSP.

Details: [`token_uri` layout and the player page](docs/token-uri-layout.md).

## Quick start for integrators

**1. Depend on the crate.** Use the Scarb and Starknet Foundry versions in [`.tool-versions`](.tool-versions).

```toml
[dependencies]
onchain_midi_player = { git = "https://github.com/Provable-Games/onchain-midi-player", tag = "v<version>" }
# Before a release tag exists: rev = "<commit>", a commit of this repository

[[target.starknet-contract]]
sierra = true
# Builds the class from the dependency so your tests can declare it.
build-external-contracts = ["onchain_midi_player::contract::TinySynth"]
```

**2. Hold a class hash.** Store it in your NFT, or in a small renderer contract the NFT calls, and let the owner change it. Take it from [`deployments/<network>.json`](deployments/sepolia.json) (`class.class_hash`). A class hash fixes the engine and the player, so tokens keep their sound until you switch. When you store or change it, check that `engine()` returns `'tinysynth'`: another engine's class shares the `midi_segment` selector and could render the wrong thing without an error.

- `version()` is the class's [SemVer](https://semver.org) version. From `1.0.0`, the major number promises call and settings-layout compatibility.
- A release has the tag `v<version>` (`release_tag` in the deployment file). A class without one is a test class: do not store its hash for production tokens. Only test classes exist so far.
- The ABIs are in [`abi/`](abi): the class's, and `ISoundProvider`'s for providers.

```cairo
use onchain_midi_player::interface::{
    ITinySynthDispatcherTrait, ITinySynthLibraryDispatcher,
};

let synth = ITinySynthLibraryDispatcher { class_hash: tinysynth_class_hash };
```

The interface, in [`src/interface.cairo`](src/interface.cairo):

| Function | Returns |
| --- | --- |
| `animation_url_segment()` | The `"animation_url":"data:text/html;base64,<page>` JSON member, pre-encoded at both base64 layers. |
| `midi_segment(midi, settings)` | This token's settings and MIDI, encoded at both layers. Reverts with `'TS: …'` on invalid settings. |
| `base64(data)` | Standard RFC 4648 base64, for your own JSON pieces. |
| `script_sha256()` | SHA-256 of the engine script. |
| `engine()` | The engine's name: `'tinysynth'`. |
| `version()` | The class's SemVer version, such as `'0.3.0'`. |
| `license()` | The license notices for the class and the code it embeds. |

**3. Assemble `token_uri`** in the layout above. [Building `token_uri` in Cairo](docs/token-uri-layout.md#building-token_uri-in-cairo) has the full function. The rules:

- Every piece you pass to `base64`, except the final `'}'`, must be a multiple of 3 bytes. Pad with spaces between JSON tokens.
- The SVG must never contain `</script`, in any letter case. See [Art (SVG) requirements](docs/token-uri-layout.md#art-svg-requirements).
- Optional: add spaces, 3 at a time, so your two largest appends start on a 31-byte `ByteArray` word. That makes them about 4x cheaper.
- `midi` and `settings` come from constants in your contract, or from a composer's [sound provider](docs/sound-provider.md).

**4. Test it.** In snforge, declare the class (never deploy it) and library-call it from your NFT. Then:

- Decode your `token_uri` and compare it with a reference, as the example's golden tests do.
- Check `engine()`, `version()` and `script_sha256()` against your class's record in [`scripts/page_versions.json`](scripts/page_versions.json), which holds the current version; for an earlier one, use `git log -p scripts/page_versions.json`.
- Test that your rendered SVGs never contain `</script`.
- Run `check-midi` on every score in CI (see below).
- Budget your largest token's gas (see [How much fits](#how-much-fits-gas-and-limits)).

[`examples/beast_consumer`](examples/beast_consumer) is a complete, tested NFT that does all of this. AI agents can use the [`integrator-guide`](plugins/onchain-midi-player/skills/integrator-guide/SKILL.md) skill.

## Quick start for composers

You need Node 22 or later and a clone of this repository. No `npm install` is needed. Use a clone whose `PAGE` matches the class you target: `grep 'pub const VERSION' src/page_data.cairo` must print the class's `version()`.

**Preview** the page a token would get, byte for byte, offline:

```sh
npm run preview -- song.mid                                       # default settings, placeholder art
npm run preview -- song.mid --settings sound.json --svg art.svg  # your settings and art
npm run preview -- song.mid --serve                               # also serve it on http://127.0.0.1:8000/
```

It checks the MIDI, the settings and the art first, and writes nothing if any check fails. Details: [Previewing a score](docs/midi-contract.md#previewing-a-score).

**Check** MIDI files against the page's own MIDI check. It exits 0 when every score passes, 1 when any fails, and 2 on a usage error, so it can gate CI:

```sh
npm run check-midi -- song.mid other.mid
```

**The MIDI contract, in short.** The class embeds the MIDI without parsing it, so a bad file does not revert: the page shows an error instead. The full rules are in the [MIDI contract](docs/midi-contract.md).

- Standard MIDI File, format 0 or 1, with ticks-per-quarter-note timing.
- End-of-Track is the last event of every track, and nothing follows the last track.
- The song always loops. A pass ends at the latest End-of-Track and lasts at least 50 ms.
- Only the tempo resets between passes. Set tempo, programs and controllers at tick 0 so every pass starts the same way.
- Channel 10 is drums: notes 35–81, and note-offs are ignored. The other channels play General MIDI programs 0–127.
- The art restarts at every pass. Make the pass a whole multiple of the art's animation periods, or the art jumps at the loop point.
- Every byte costs gas: about 14M L2 gas per KB.

**Sound settings.** Each `midi_segment` call takes a `TinySynthSettings` value, declared in [`src/types.cairo`](src/types.cairo):

- `quality`: the TinySynth engine's built-in sounds, 0 chip-tune or 1 FM.
- `reverb`, `master_vol` and `voices`: the mix and the polyphony.
- `timbres`: custom sounds. Each replaces a General MIDI program (0–127) or a drum note (35–81), so the MIDI selects it as usual.
- `waves`: custom waveforms (sample tables or harmonics) that timbres can use. Operators can also carry a fixed low-, high- or band-pass filter.

Fractional values are fixed point, in units of 1/10,000. Invalid settings revert with a `'TS: …'` message and the offending indices. The reference is [Sound settings](docs/sound-settings.md).

**Serving sound from a contract.** A composer's contract can serve each token's MIDI and settings in one call, through `ISoundProvider` in [`src/interface.cairo`](src/interface.cairo). The NFT calls it and passes the result to `midi_segment`.

```cairo
#[starknet::interface]
pub trait ISoundProvider<T> {
    fn get_sound(self: @T, token_id: u256) -> TinySynthSound; // { midi: ByteArray, settings: TinySynthSettings }
}
```

`get_sound` takes the token ID as minted, returns a raw MIDI file and valid settings, and is deterministic and view-only. Each engine has its own typed settings struct, with no enum, and the layouts of `TinySynthSettings` and `TinySynthSound` are fixed at release. A later engine's providers will implement `get_<engine>_sound`; `get_sound` belongs to the TinySynth engine. The rules, and a safe way to call a provider, are in [Sound provider interface](docs/sound-provider.md).

AI agents can use the [`midi-guide`](plugins/onchain-midi-player/skills/midi-guide/SKILL.md) and [`sound-design`](plugins/onchain-midi-player/skills/sound-design/SKILL.md) skills.

## How much fits (gas and limits)

A marketplace or indexer reads `token_uri` with a view call (`starknet_call`). The RPC node that serves the call caps its gas, so your largest token must fit under the caps of the nodes your users rely on. Estimate a token in L2 gas (Sierra gas, as snforge reports it by default):

| Part of `token_uri` | L2 gas |
| --- | --- |
| The fixed page: `animation_url_segment()` and appending it, word-aligned | about 10M |
| Art: per KB of SVG (it is encoded twice) | about 9M |
| MIDI: per KB | about 14M |
| `SETTINGS`: per KB | about 14.5M |
| Your contract's own work: rendering the SVG, the JSON members | yours to measure |

`SETTINGS` is the text form of your `TinySynthSettings`. The defaults are 16 bytes, and three typical custom sounds about 0.3 KB. Each operator adds about 50 bytes, and each wave sample or harmonic 2 to 6 bytes. `SETTINGS` has no size cap: gas and the node's cap are the only limits.

**Example.** A full-size token from [`examples/beast_consumer`](examples/beast_consumer) costs **288.6M** L2 gas: a 22,733-byte animated SVG, a 3,716-byte MIDI file and 334 bytes of `SETTINGS`. A token with a 1 KB SVG and a 112-byte MIDI file costs about 33M.

**Node caps** on a view call's gas:

| Node | Cap |
| --- | --- |
| Pathfinder | 10B |
| Madara | 10B |
| Katana (development) | 1B by default |
| Juno | 100M by default (`--rpc-call-max-gas`) |
| Hosted providers | Undocumented. PublicNode refused the full-size example |

- Check a full-size token through the providers your marketplaces and indexers use.
- At 10B, a token has room for about 670 KB of `SETTINGS`.
- Nodes built on jsonrpsee cap responses at 10 MiB, a `token_uri` of about 4.85 MB.
- If another contract calls `token_uri` inside a transaction, the transaction cap of 1.11B L2 gas applies.
- Keep every class in the call chain at Sierra 1.7 or later. An older class switches its frame to Cairo-steps accounting, capped at 10M steps (Juno: 4M).

Measurements, per-function costs and the sources of the caps: [Gas and limits](docs/gas.md).

## Verifying the engine

Every token's `animation_url` carries the engine, so anyone can check it offline:

1. Save the collection's `token_uri` (from its contract, an explorer or a marketplace) to `token_uri.txt`.
2. Run `node scripts/verify_engine.mjs token_uri.txt --expect <script_sha256>`. It needs Node 22 or later and nothing else. It prints the SHA-256 of the engine, of its gzip payload and of the fixed page.
3. Compare them with the record for the class's `version()` in [`scripts/page_versions.json`](scripts/page_versions.json) (`script_sha256`, `gzip_sha256`, `page_sha256`). For an earlier version, find its record with `git log -p scripts/page_versions.json`.
4. Optionally, rebuild the engine from the fork commit in that record (`engine_commit`), and the page and class from this repository.

Shell and Python versions of step 2, and the rebuild steps: [Verifying the engine](docs/verifying.md).

## Agent skills

Four skills help AI agents in other projects use the player. They live in [`plugins/onchain-midi-player/skills/`](plugins/onchain-midi-player/skills) and link to these docs.

| Skill | For |
| --- | --- |
| [`integrator-guide`](plugins/onchain-midi-player/skills/integrator-guide/SKILL.md) | Adding the player to a contract's `token_uri`, testing it and budgeting its gas |
| [`midi-guide`](plugins/onchain-midi-player/skills/midi-guide/SKILL.md) | Writing MIDI for the player, previewing it and keeping it in step with the art |
| [`sound-design`](plugins/onchain-midi-player/skills/sound-design/SKILL.md) | Designing `TinySynthSettings`: custom timbres, waves and filters |
| [`token-uri-inspector`](plugins/onchain-midi-player/skills/token-uri-inspector/SKILL.md) | Decoding, verifying and viewing a deployed or local `token_uri` |

**Install in Claude Code.** This repository is a plugin marketplace ([`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json)). In your project:

```sh
claude plugin marketplace add Provable-Games/onchain-midi-player    # or Provable-Games/onchain-midi-player#<tag> to pin a ref
claude plugin install onchain-midi-player@onchain-midi-player --scope project
```

The skills then run as `/onchain-midi-player:midi-guide` and so on. `claude plugin update onchain-midi-player@onchain-midi-player` brings the latest. To offer them to everyone who opens your project, commit this to its `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "onchain-midi-player": { "source": { "source": "github", "repo": "Provable-Games/onchain-midi-player" } }
  },
  "enabledPlugins": { "onchain-midi-player@onchain-midi-player": true }
}
```

**Install in Codex.** Ask the built-in [`$skill-installer`](https://learn.chatgpt.com/docs/build-skills#install-curated-skills-for-local-use) to install all four skills. Send this prompt in Codex:

```text
$skill-installer Install these skills from Provable-Games/onchain-midi-player:
- plugins/onchain-midi-player/skills/integrator-guide
- plugins/onchain-midi-player/skills/midi-guide
- plugins/onchain-midi-player/skills/sound-design
- plugins/onchain-midi-player/skills/token-uri-inspector
```

For a pinned version, add `Use ref <tag-or-commit> for all four skills` to the prompt. The installer adds them to your user skills directory, available across projects. Open `/skills` to check they appear; restart Codex if needed. Invoke them as `$integrator-guide`, `$midi-guide`, `$sound-design` or `$token-uri-inspector`.

For a project-local installation, run this from your project's root, using the path to your clone of this repository:

```sh
mkdir -p .agents/skills
cp -R /path/to/onchain-midi-player/plugins/onchain-midi-player/skills/. .agents/skills/
```

Copy all four folders, including their scripts and references, to preserve links between skills. Commit `.agents/skills/` to share them with your team. To update a project-local installation, repeat the copy from an updated clone. For an installer-managed update, remove the installed skill folders before asking `$skill-installer` to install them again from the desired ref.

**Other agents.** Each `SKILL.md` follows the open [Agent Skills](https://agentskills.io/specification) format. Copy the whole `skills/` folder into your agent's skills directory to keep the links between skills.

The tools the skills use need Node 22 or later and a clone whose `VERSION` matches your class (see [Quick start for composers](#quick-start-for-composers)). Every commit with the same `VERSION` has the same `PAGE`, so use `main` while its `VERSION` matches, otherwise the last commit before it changed (`git log --oneline -- src/page_data.cairo`). A released class's tag, `v<version>`, works too.

## Documentation

- [`token_uri` layout and the player page](docs/token-uri-layout.md)
- [MIDI contract](docs/midi-contract.md)
- [Sound settings](docs/sound-settings.md)
- [Sound provider interface](docs/sound-provider.md)
- [Gas and limits](docs/gas.md)
- [Verifying the engine](docs/verifying.md)
- [`deployments/`](deployments): what is declared and deployed on each network, one JSON file per network
- [`abi/`](abi): the ABIs of the class and of `ISoundProvider`
- [`scripts/page_versions.json`](scripts/page_versions.json): the current `version()`'s engine commit, page revision and hashes (earlier versions are in its git history)
- [Development](docs/development.md): toolchain, build, tests and CI, for contributors

## Credits
  - **Casey Wescott** ([@caseywescott](https://github.com/caseywescott)): co-designer of onchain-midi-player

## License

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE). The embedded TinySynth engine is also Apache-2.0: copyright Tatsuya Shinyagaito (g200kg), modified by Provable Games in <https://github.com/Provable-Games/webaudio-tinysynth>. The page's gunzip shim is derived from fflate, MIT License, Copyright (c) 2026 Arjun Barrett ([text](tests/vendor/fflate-0.8.3.LICENSE)). The class's base64 encoder is the `game_components_encoding` package of [game-components](https://github.com/Provable-Games/game-components), MIT License, Copyright (c) 2026 Provable Games ([text](tests/vendor/game-components.LICENSE)). `license()` includes all four notices.

The Beast and MIDI test fixtures are Apache-2.0 as well. The Beast SVG in [`tests/fixtures/beasts/`](tests/fixtures/beasts/README.md) is Beasts artwork that Provable Games licenses under Apache-2.0 for this repository, and the MIDI scores in [`tests/fixtures/midi/`](tests/fixtures/midi/README.md) are synthetic, generated by `scripts/gen_midi_fixtures.mjs`. Neither is part of the class.
