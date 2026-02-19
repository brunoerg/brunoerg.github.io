# Gocoin incorrectly parsing `OP_PUSHDATA4`

Gocoin is a full Bitcoin solution written entirely from scratch in Go language with zero dependencies, according to their website. We are differentially fuzzing its `VerifyTxScript` function against Bitcoin Core's `VerifyScript`. These functions are consensus critical and they are responsible for verifying `scriptPubKey` and `scriptSig`.

During the execution, we got a crash with the following scripts:

```sh
scriptSig: 5b00
scriptPubKey: 00004c020f600100009f9a000102020202004e000064a2a2a2a2a2000000a2a2a2a2a2a2a2a2 
```

The error returned by Bitcoin Core is `SCRIPT_ERR_BAD_OPCODE` that was being returned due to `OP_PUSHDATA4` claiming more bytes than are available. 

`GetOp()` attempts to advance pc by 100 bytes, overshoots the end of the script buffer, and then, returns false. Looking at `gocoin`'s code, the similar function to Core's `GetOpcode` is `GetOp`:

```go
func GetOpcode(b []byte) (opcode int, ret []byte, pc int, e error) {
	// Read instruction
	if pc+1 > len(b) {
		e = errors.New("GetOpcode error 1")
		return
	}
	opcode = int(b[pc])
	pc++

	if opcode <= OP_PUSHDATA4 {
		size := 0
		if opcode < OP_PUSHDATA1 {
			size = opcode
		}
		if opcode == OP_PUSHDATA1 {
			if pc+1 > len(b) {
				e = errors.New("GetOpcode error 2")
				return
			}
			size = int(b[pc])
			pc++
		} else if opcode == OP_PUSHDATA2 {
			if pc+2 > len(b) {
				e = errors.New("GetOpcode error 3")
				return
			}
			size = int(binary.LittleEndian.Uint16(b[pc : pc+2]))
			pc += 2
		} else if opcode == OP_PUSHDATA4 {
			if pc+4 > len(b) {
				e = errors.New("GetOpcode error 4")
				return
			}
			size = int(binary.LittleEndian.Uint16(b[pc : pc+4]))
			pc += 4
		}
		if pc+size > len(b) {
			e = errors.New(fmt.Sprint("GetOpcode size to fetch exceeds remainig data left: ", pc+size, "/", len(b)))
			return
		}
		ret = b[pc : pc+size]
		pc += size
	}

	return
}
```

The issue is that gocoin was using `Uint16` in the context of `OP_PUSHDATA4` instead of using `Uint32`. The size field is explicitly defined as a 4-byte little-endian unsigned integer.


--------------------------------------

Thanks Piotr Narewski for responding to my report and fixing the bug so quickly. Also, he told me to go ahead and disclosure it.



