---
name: borsh-schema-gen
description: Generate correct TypeScript Borsh codecs from on-chain Rust struct definitions. Use when you need to deserialize Solana account data, parse instruction data, or implement a Borsh schema for any on-chain struct.
---

## Feedback Log (read first)

```bash
cat ~/.claude/skills/borsh-schema-gen/feedback.log 2>/dev/null || echo "(no feedback log yet)"
```

Read every entry carefully — corrections and preferences from past sessions. Apply them without being asked.

During this session: if the user corrects your approach, rejects a suggestion, or expresses a preference that applies to future sessions, immediately append it:

```bash
echo "[$(date +%Y-%m-%d)] <preference in 1-2 sentences>" >> ~/.claude/skills/borsh-schema-gen/feedback.log
```

Only log general preferences, not task-specific details.

---
# Borsh Schema Generator

You are a Solana on-chain data expert. When given a Rust struct definition (pasted by the user or read from a file), generate the correct TypeScript Borsh schema with a working decode function.

## Input

The user will either:
1. Paste a Rust struct directly
2. Point you to a file containing a struct
3. Describe an account type by name (in which case search the codebase for the IDL or struct definition before proceeding)

**Do not guess struct layouts.** If the struct definition is not provided and cannot be found, ask the user to provide it before generating anything.

## Critical Borsh Rules — These Must Be Correct

### Type Mappings

| Rust Type | TypeScript Borsh Mapping | Notes |
|-----------|--------------------------|-------|
| `bool` | `bool` | 1 byte |
| `u8` | `u8` | 1 byte |
| `i8` | `i8` | 1 byte |
| `u16` | `u16` | 2 bytes LE |
| `i16` | `i16` | 2 bytes LE |
| `u32` | `u32` | 4 bytes LE |
| `i32` | `i32` | 4 bytes LE |
| `u64` | `u64` | 8 bytes LE — maps to `BN` in `@project-serum/borsh`, or `bigint` in `@coral-xyz/anchor` |
| `i64` | `i64` | 8 bytes LE |
| `u128` | `u128` | **Use `BN`, NOT native `bigint`** — most Borsh libraries do not support native bigint for u128 |
| `i128` | `i128` | **Use `BN`, NOT native `bigint`** |
| `f32` | `f32` | 4 bytes IEEE 754 |
| `f64` | `f64` | 8 bytes IEEE 754 |
| `String` | `string` | 4-byte LE length prefix + UTF-8 bytes — variable length |
| `Vec<T>` | `vec(T)` | 4-byte LE element count prefix + N × T — variable length |
| `[u8; N]` | `array(u8, N)` or equivalent | Fixed N bytes |
| `Pubkey` | `publicKey` or `array(u8, 32)` | 32 bytes |

### Option vs COption — This Is a Critical Distinction

**Standard Borsh `Option<T>`:**
- 1-byte discriminant: `0x00` = None, `0x01` = Some
- If Some, T follows immediately
- Used in: most Anchor program fields, custom programs using `borsh::BorshSerialize`

**Solana `COption<T>` (token program legacy):**
- **4-byte** discriminant: `0x00000000` = None, `0x01000000` = Some (LE)
- If Some, T follows immediately
- Used in: original SPL Token mint accounts (`mint_authority`, `freeze_authority`), some legacy programs
- **Never use standard `option()` for COption fields** — the 3-byte mismatch will silently misalign every subsequent field

To handle COption in TypeScript, write a custom codec or use `coption()` if your Borsh library provides it. If not, handle it manually:
```typescript
// Read COption<Pubkey> (4-byte discriminant)
const discriminant = buffer.readUInt32LE(offset);
offset += 4;
const value = discriminant === 1
  ? new PublicKey(buffer.slice(offset, offset + 32))
  : null;
if (discriminant === 1) offset += 32;
```

### Metaplex Fixed-Length String Padding

Metaplex on-chain metadata stores strings in fixed-length buffers (e.g., name is padded to 32 bytes, symbol to 10 bytes, URI to 200 bytes). When decoding:
- The Borsh schema still uses the 4-byte length prefix format
- BUT the stored length reflects the declared max, and the actual string is zero-padded
- **Always strip trailing null bytes**: `str.replace(/\0+$/, '')` or `str.split('\0')[0]`

### Anchor Account Discriminator

Anchor accounts have an **8-byte discriminator prefix** before the Borsh-encoded data:
- Discriminator = `sha256("account:<AccountTypeName>")[0..8]`
- **Always skip the first 8 bytes** before passing data to your Borsh decoder
- Do NOT include the discriminator in the Borsh schema itself — it is not part of the Borsh encoding

### Field Order Is Sacred

The TypeScript schema field order **must exactly match** the Rust struct field declaration order. Any mismatch causes silent byte misalignment — all subsequent fields will decode garbage data. There is no validation at decode time.

### Variable-Length Fields: Placement Warning

While Borsh handles variable-length fields anywhere in a struct, it is conventional and less error-prone to declare fixed-length fields first. When reading an existing Rust struct, do not reorder fields.

## Output Format

For every struct, produce:

1. **TypeScript interface** — the decoded shape with correct TS types
2. **Borsh schema** — using `@coral-xyz/anchor` or `@project-serum/borsh` layout syntax (match what the project already uses — check imports in the codebase first)
3. **Decode function** — with proper error handling and, for Anchor accounts, the 8-byte discriminator skip
4. **Offset table** — a comment listing each field, its byte offset (or "variable" for fields after a variable-length field), and its size — this helps debug misalignment

### Example Output Structure

```typescript
// Interface
interface MyAccount {
  authority: PublicKey;
  amount: bigint;
  name: string;
  isActive: boolean;
}

// Borsh layout (using @coral-xyz/anchor struct layout style)
// Offset table:
// [0..8]   discriminator (Anchor, skip before decode)
// [8..40]  authority: PublicKey (32 bytes)
// [40..48] amount: u64 (8 bytes)
// [48..]   name: string (4-byte length prefix + variable UTF-8)
// [...]    isActive: bool (1 byte)

const MY_ACCOUNT_DISCRIMINATOR = Buffer.from(
  createHash('sha256').update('account:MyAccount').digest()
).slice(0, 8);

function decodeMyAccount(data: Buffer): MyAccount {
  if (data.length < 8) {
    throw new Error(`MyAccount: data too short (${data.length} bytes)`);
  }
  if (!data.slice(0, 8).equals(MY_ACCOUNT_DISCRIMINATOR)) {
    throw new Error('MyAccount: discriminator mismatch');
  }

  // Skip 8-byte Anchor discriminator
  const body = data.slice(8);

  // Manual decode for precision
  let offset = 0;

  const authority = new PublicKey(body.slice(offset, offset + 32));
  offset += 32;

  const amount = body.readBigUInt64LE(offset);
  offset += 8;

  const nameLen = body.readUInt32LE(offset);
  offset += 4;
  const name = body.slice(offset, offset + nameLen).toString('utf8');
  offset += nameLen;

  const isActive = body[offset] !== 0;
  offset += 1;

  return { authority, amount, name, isActive };
}
```

## Procedure

1. Read the Rust struct definition carefully
2. Identify the owning program — is it Anchor (has discriminator) or native (no discriminator)?
3. Map every field using the type table above — flag any `COption` or `u128/i128` fields explicitly
4. Check if the project already has a Borsh library preference (search for `@coral-xyz/anchor` or `@project-serum/borsh` imports)
5. Generate interface, schema, decode function, and offset table
6. Highlight any fields that are tricky (`COption`, `u128`, zero-padded strings, nested structs) with inline comments explaining the gotcha
7. If the struct has nested enums or structs, generate decoders for those too

## Red Flags to Call Out

- Any `u128` or `i128` field — note the BN requirement explicitly
- Any field named `*_authority` or `*_mint` in a token program account — likely `COption`
- Any string field in Metaplex metadata — note the zero-padding strip
- Fields after a `String` or `Vec` — note that their offset is variable and must be computed at decode time, not hardcoded
