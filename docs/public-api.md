# Public API

What an application can import from RoomKit and rely on, and how a change
that breaks it is announced.

## What is public

A name is public when the `__all__` of `roomkit` or of one of its packages
(an `__init__.py`) lists it, and the application imports it from that
package:

```python
from roomkit import RoomKit, TextContent, HookTrigger
from roomkit.channels import SMSChannel, VoiceChannel
from roomkit.voice import AudioPipelineConfig
from roomkit.voice.stt import DeepgramSTTProvider, DeepgramConfig
from roomkit.voice.backends import LocalAudioBackend
from roomkit.providers.anthropic import AnthropicAIProvider, AnthropicConfig
```

A name can be public in more than one package: `TextContent` is listed by
both `roomkit` and `roomkit.models`. Either import is public.

A provider whose module needs an optional dependency is listed by its
package all the same. Its module loads the first time the name is used, so
`import roomkit` never needs the extra; when the extra is missing, that first
use raises `ModuleNotFoundError` naming the package to install.

## What is internal

- **Module paths.** `roomkit.models.event`, `roomkit.core.framework` and
  `roomkit.voice.stt.deepgram` are where the code lives, not where it is
  imported from. They import today, but a release may move, merge or rename
  them without a BREAKING entry, even when they hold a public name.
- **Underscore names**, in any package or module.
- **Lookup helpers that predate the package exports**, such as the
  `get_deepgram_provider()` family on `roomkit.voice`. They keep working;
  import the class from its package instead.

The pages of this site, the examples in the repository, its README and the
`llms.txt` files import only public paths, and a test keeps them on it. The
API reference shows each public name under the path to import it from, or
writes that path above the entry.

## What counts as a break

A change breaks the public API when code that worked against the previous
release can stop working:

- a name removed from the `__all__` that listed it;
- a call that worked and fails: a parameter removed or renamed, made
  required, moved to another position, made keyword-only, or its default
  changed; a method made `async`, or a property made a method;
- an enum member removed or its value changed;
- a documented behaviour changed.

A new name, a new parameter with a default, a new enum member, and any
change to an internal module are not breaks.

## How a break is announced

Each break is a **BREAKING** entry in
[CHANGELOG.md](https://github.com/roomkit-live/roomkit/blob/main/CHANGELOG.md),
in the release that ships it, saying what changed and what to write
instead. Until 1.0, a release that carries one is a minor release (`0.N.0`);
a patch release (`0.N.x`) carries none.

Two files in the repository hold the public surface: every name each
package exports, and the signature of every public class and function. A
change that drops a name or breaks a call fails the test suite until the
commit that makes it rewrites them, and that commit carries the BREAKING
entry.
