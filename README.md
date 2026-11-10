# codeassist-dex

Pre-dexed and pre-indexed Android libraries for [CodeAssist](https://github.com/tyron12233/CodeAssist), laid out
like Maven Central but holding dex and editor index segments instead of jars.

When CodeAssist builds an app on a phone, converting the app's libraries to dex (D8) used to be most of a new
project's first build: about 85 seconds of a 97-second Compose build on a 2 GB device. With this repository the
phone downloads each library's finished dex instead, and the same build took about 21 seconds. The editor
likewise downloads each library's index segments (what code completion and navigation search) instead of
indexing the library on the phone. The phone only downloads the library versions its project uses, and the
"Download prebuilt libraries" setting (Automatically / Wi-Fi only / Off) controls it.

## Layout

```
dex/<flavor>/<group path>/<artifact>/<version>/<artifact>-<version>.dex.zip
```

for example `dex/v2-r8-8.13.19/minApi26-debug-merged/androidx/compose/ui/ui-android/1.11.2/ui-android-1.11.2.dex.zip`.

The flavor names the D8 build that made the dex and how: a CodeAssist release with a different D8 reads and
writes its own flavor, so files from different D8 versions never mix. Dex is built for minSdk 26 and runs on
every later Android version; projects below minSdk 26 dex their libraries on the phone as before.

Each archive holds the merged dex of every jar the library version ships (one for a plain jar; the
`classes.jar` and any `libs/` jars of an AAR), plus a manifest naming each jar's content hash.

Index segments live next to them:

```
index/<flavor>/<group path>/<artifact>/<version>/<artifact>-<version>.index.zip
```

The flavor is the segment format plus a digest of the IDE's indexers and their versions, so a CodeAssist release
that changes an indexer gets a flavor of its own. Each archive holds `<index id>/v<index version>/<key>.seg` for
every index the IDE runs: a jar's segments (`jar-<content id>`), for an AAR its `res/` folder's
(`res-<AAR content id>`), and the version's `-sources.jar` (`src-<content id>`: parameter names and javadoc). A phone installs only
the indexes, and versions, it runs itself; an index that changed version is rebuilt on the phone, like
IntelliJ's shared indexes. Segments are keyed by the jar's content, so the same jar in any project or on any
device uses the same segments.

## Trust

Every archive is signed. Its manifest lists the SHA-256 of every file in it and is signed with an ECDSA P-256
key that only the publisher holds; the public half is built into CodeAssist. A phone installs an archive only
when:

- the signature verifies and every file matches the manifest, with nothing extra;
- the manifest names the exact content hash of the jar the phone resolved (a republished or patched artifact
  is dexed on the phone instead);
- the dex defines every class that jar contains (for dex), or the index segments are keyed by that jar's
  content (for indexes).

Anything else is ignored and the library is dexed on the phone.

## Adding libraries

Add a line to [`libraries.txt`](libraries.txt): `group:artifact` (its 3 newest stable releases), a full
coordinate (`group:artifact:version`), or a CodeAssist project template (`template:<id>`). On push, and every Monday, the [publish workflow](.github/workflows/publish.yml) resolves each
line in a fresh project, dexes and indexes every library version in its dependency closure that isn't published
yet, signs it and commits it. Published versions are never rewritten. The weekly run also picks up new library releases,
and a CodeAssist release with a new D8 or new indexers, whose flavors it fills in from scratch.

To publish by hand with the CodeAssist CLI:

```
codeassist dexrepo --repo . --key <signing key> --libraries libraries.txt
```
