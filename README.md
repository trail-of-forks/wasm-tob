# wasm-tob

Python module capable of decoding and disassembling WebAssembly modules
and bytecode, according to the MVP specification of the WASM binary
format.

As there is no official text format defined yet, the text format
implemented doesn't correspond to any existing definition and is a
simple `mnemonic op1, op2, ...` format. Functions are formatted in a
way similar to how Google Chrome does in the debug console.

## ❗ Important

This is a fork of the original project that the author was no longer able to
spend time on: https://github.com/athre0z/wasm. The changes made here are
primarily to support the
[Manticore](https://github.com/trailofbits/manticore) project.

New issues and pull requests will be reviewed on a best-effort basis. Please
open an issue first if you think fixing the problem will be complex; this is so
we can evaluate whether a fix or feature is in scope before comitting time to
review. When opening an issue, please include information on how to reproduce
what you are seeing. If you feel comfortable, please submit a well-crafted,
minimal pull request that we can review.

## Installation

```sh
# From PyPi
pip install wasm-tob

# From GitHub
pip install git+https://github.com/trail-of-forks/wasm-tob.git
```

## Examples

Parsing a WASM module, printing the types of sections found.

```python
from wasm_tob import decode_module

with open('input-samples/hello/hello.wasm', 'rb') as raw:
    raw = raw.read()

mod_iter = iter(decode_module(raw))
header, header_data = next(mod_iter)

for cur_sec, cur_sec_data in mod_iter:
    print(cur_sec_data.get_decoder_meta()['types']['payload'])
```

Possible output:

```text
<wasm_tob.modtypes.TypeSection object at 0x108249b90>
<wasm_tob.modtypes.ImportSection object at 0x108249bd0>
<wasm_tob.modtypes.FunctionSection object at 0x108249c10>
<wasm_tob.modtypes.GlobalSection object at 0x108249cd0>
<wasm_tob.modtypes.ExportSection object at 0x108249d10>
<wasm_tob.modtypes.ElementSection object at 0x108249d90>
<wasm_tob.modtypes.CodeSection object at 0x108249dd0>
<wasm_tob.modtypes.DataSection object at 0x108249e10>
<wasm_tob.types.BytesField object at 0x108249b10>
```

Parsing specific sections (eg. GlobalSection, ElementSection, DataSection) in WASM module, printing each section's content:

```python
from wasm_tob import (
    decode_module,
    format_instruction,
    format_lang_type,
    format_mutability,
    SEC_DATA,
    SEC_ELEMENT,
    SEC_GLOBAL,
)

with open('input-samples/hello/hello.wasm', 'rb') as raw:
    raw = raw.read()

mod_iter = iter(decode_module(raw))
header, header_data = next(mod_iter)

for cur_sec, cur_sec_data in mod_iter:
    if cur_sec_data.id == SEC_GLOBAL:
        print("GlobalSection:")
        for idx, entry in enumerate(cur_sec_data.payload.globals):
            print(
                format_mutability(entry.type.mutability),
                format_lang_type(entry.type.content_type),
            )

            for cur_insn in entry.init:
                print(format_instruction(cur_insn))

    if cur_sec_data.id == SEC_ELEMENT:
        print("ElementSection:")
        for idx, entry in enumerate(cur_sec_data.payload.entries):
            print(entry.index, entry.num_elem, entry.elems)
            for cur_insn in entry.offset:
                print(format_instruction(cur_insn))

    if cur_sec_data.id == SEC_DATA:
        print("DataSection:")
        for idx, entry in enumerate(cur_sec_data.payload.entries):
            print(entry.index, entry.size, entry.data.tobytes())
            for cur_insn in entry.offset:
                print(format_instruction(cur_insn))
```

Output:

```text
GlobalSection:
mut i32
get_global 0
end
mut i32
get_global 1
end
[...]
mut f32
f32.const 0x0
end
mut f32
f32.const 0x0
end
ElementSection:
0 12576 [856, 856, 856, [...], 888]
i32.const 0
end
DataSection:
0 16256 b'\x98&\x00\x00\xfe4\x00\x00\x10\x04\x00\x00\x00...\x00N10__cxxabiv121__vmi_class_type_infoE'
get_global 8
end

```

Manually disassemble WASM bytecode, printing each instruction.

```python
from wasm_tob import (
    decode_bytecode,
    format_instruction,
    INSN_ENTER_BLOCK,
    INSN_LEAVE_BLOCK,
)

raw = bytearray([2, 127, 65, 24, 16, 28, 65, 0, 15, 11])
indent = 0
for cur_insn in decode_bytecode(raw):
    if cur_insn.op.flags & INSN_LEAVE_BLOCK:
        indent -= 1
    print('  ' * indent + format_instruction(cur_insn))
    if cur_insn.op.flags & INSN_ENTER_BLOCK:
        indent += 1
```

Output:

```text
block -1
  i32.const 24
  call 28
  i32.const 0
  return
end
```

## `wasmdump` command-line tool

The module also comes with a simple command-line tool called `wasmdump`,
dumping all module struct in tree format. Optionally, it also
disassembles all functions found when invoked with `--disas` (slow).

## Version support

This project aims to support all Python releases that are still actively
supported and maintained. If you encounter issues with a particular Python
version, please open an issue.
