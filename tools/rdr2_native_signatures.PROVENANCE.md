# `rdr2_native_signatures.h`

Vendored verbatim from
<https://github.com/rfoodxmodz/PS4-RDR2-Native-Invoker>
(`rdrPayload/include/natives.h`).

The upstream MIT license is retained in `rdr2_native_signatures.LICENSE`.

It is used **only** for the `name -> hash -> C signature` mapping. Its own
address tables are ignored: they cover game versions 1.00 through 1.29, and the
build we target is **1.32**, whose handler addresses are different. Checked
rather than assumed — every address in its `nativesFunctions_129[]` lands
mid-instruction in our eboot.

The reason a file written against 1.29 is usable at all is that **RDR2 native
hashes are version-stable**: comparing its own per-version hash arrays gives
98–99% set overlap across 1.13/1.19/1.24/1.29, with the differences being
natives added or removed rather than re-keyed. The hash for a given name does
not move between console patches; only the address does. That is what makes
`tools/gen_natives.py` a join of *this file's names* against *our eboot's
addresses*, and it is why RDR2Lua needs none of the ordered-list alignment that
gave GTALua eleven off-by-one bindings.

Corroboration that the two halves belong together: 2,063 of the 2,069 hashes
this file knows (union over all its versions) appear in the registry extracted
from our own eboot — 99.7%.
