# Solana: How to send custom instructions via instruction data

In this article, we'll walk through the process of sending a custom instruction to a solana on-chain program. We'll modify the solana [example helloworld][1] to take two instructions, `SayHello` and `SayGoodbye`.

Here's a look at the final result:
![output from ](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/37ke6vjrdsga2mc61cao.png)

Let's get started. Picking up from where the [helloworld][1] tutorial leaves us.

## Define API of the program

```rust
// instruction.rs
#[derive(Debug)]
pub enum HelloInstruction {
    SayHello,
    SayBye,
}
```

We create an `enum` that will contain what operations our program is able to perform.

`instruction.rs` is responsible for decoding `instruction_data`.

## Decode `instruction_data`

```diff
// instruction.rs
+use solana_program::{program_error::ProgramError};
+
#[derive(Debug)]
pub enum HelloInstruction {
    SayHello,
    SayBye,
}

+impl HelloInstruction {
+    pub fn unpack(input: &[u8]) -> Result<Self, ProgramError> {
+        let (&tag, rest) = input
+            .split_first()
+            .ok_or(ProgramError::InvalidInstructionData)?;
+
+        Ok(match tag {
+            0 => HelloInstruction::SayHello,
+            1 => HelloInstruction::SayBye,
+            _ => return Err(ProgramError::InvalidInstructionData),
+        })
+    }
+}
```

We create an associated function called `unpack` which takes the `instruction_data` and returns either a variant of the `HelloInstruction` or a `ProgramError`.

```rust
// --snip--
let (&tag, rest) = input
    .split_first()
    .ok_or(ProgramError::InvalidInstructionData)?;
// --snip--
```

We split the `instruction_data` slice and receive a tuple containing `&tag` which is the first element and `rest` which are the remaining bytes.

`&tag` (a number/u8 ranges; 0 - 255) is mapped 1:1 to an operation. This becomes clearer when we create the instruction on the client, more on that later on.

`rest` will have the remaining bytes that may contain additional information that the operation will need.

We transform the `Option` with `.ok_or(ProgramError::InvalidInstructionData)?;` into a `Result`, if the `Option` is of the `None` variant we return an `InvalidInstructionData` error. More on the [ok_or][2] method.

```rust
Ok(match tag {
    0 => HelloInstruction::SayHello,
    1 => HelloInstruction::SayBye,
    _ => return Err(ProgramError::InvalidInstructionData),
})
```

We can now match the `tag` to know what operation our program should run. We return an error if none of the known instructions are matched.

## Update `process_instruction`

```diff
// lib.rs
// --snip--
+pub mod instruction;
+use crate::instruction::HelloInstruction;
// --snip--

pub fn process_instruction(
    program_id: &Pubkey, // Public key of the account the hello world program was loaded into
    accounts: &[AccountInfo], // The account to say hello to
    instruction_data: &[u8],
) -> ProgramResult {
    msg!("Hello World Rust program entrypoint");
+
+   let instruction = HelloInstruction::unpack(instruction_data)?;
+   msg!("Instruction: {:?}", instruction);

    // Iterating accounts is safer then indexing
    let accounts_iter = &mut accounts.iter();

    // Get the account to say hello to
    let account = next_account_info(accounts_iter)?;

    // The account must be owned by the program in order to modify its data
    if account.owner != program_id {
        msg!("Greeted account does not have the correct program id");
        return Err(ProgramError::IncorrectProgramId);
    }

    // Increment and store the number of times the account has been greeted
    let mut greeting_account = GreetingAccount::try_from_slice(&account.data.borrow())?;
    greeting_account.counter += 1;
    greeting_account.serialize(&mut &mut account.data.borrow_mut()[..])?;

    msg!("Greeted {} time(s)!", greeting_account.counter);

    Ok(())
}
```

We bring our `HelloInstruction` into scope with `use crate::instruction::HelloInstruction;` and attempt to decode the instruction with `let instruction = HelloInstruction::unpack(instruction_data)?;`

For now we'll just log the instruction. Later on we'll modify our code to use this instruction.

A quick recap of what we've done so far:

1. Defined our program API with the `HelloInstruction` `enum`
2. Unpacked (aka deserialize) our `instruction_data` byte array to retrieve the operation from our client
3. Logged the decoded instruction

Let's `build` and `re-deploy` our program. If you're unsure how to do this, take a look at the [example helloworld docs][1]. We can now run our client and see what our log message shows.
![output from solana logs](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ql84n6reb1l0uuvgn671.png)

We get an error: `Program log: Instruction Err(InvalidInstructionData)` because our client is not passing a valid instruction. Let's fix that.

## Update `sayHello`

```diff
// hello_world.ts
// --snip --

/**
 * Say hello
 */
export async function sayHello(): Promise<void> {
  console.log('Saying hello to', greetedPubkey.toBase58());
  const instruction = new TransactionInstruction({
    keys: [{pubkey: greetedPubkey, isSigner: false, isWritable: true}],
    programId,
-   data: Buffer.alloc(0),
+   data: createSayHelloInstructionData(),
  });
  await sendAndConfirmTransaction(
    connection,
    new Transaction().add(instruction),
    [payer],
  );
}
```

We replace the call to `Buffer.alloc(0)` with our new function `createSayHelloInstructionData()` which will return a `Buffer`.

## Create `createSayHelloInstructionData`

For this we'll need to install two libraries.

```sh
npm i @solana/buffer-layout buffer
```

```typescript
// hello_world.ts

import * as BufferLayout from '@solana/buffer-layout';
import { Buffer } from 'buffer';

// --snip--

function createSayHelloInstructionData(): Buffer {
  const dataLayout = BufferLayout.struct([
    BufferLayout.u8('instruction')
  ]);

  const data = Buffer.alloc(dataLayout.span);
  dataLayout.encode({
    instruction: 0
  }, data);

  return data;
}
```

We create a buffer layout structure with one field; `instruction`. This is where we'll encode what operation we want to run in our program, which is a `u8`.

```typescript
const data = Buffer.alloc(dataLayout.span);
```

We then allocate a new buffer using the size from the buffer layout that we created earlier.

```typescript
dataLayout.encode({
  instruction: 0
}, data);
```

Finally we encode the instruction (`instruction: 0`) into the buffer.

Th `instruction: 0` corresponds to the `tag` variable that we get in the first element of the `instruction_data` slice.

```rust
// --snip--
let (&tag, rest) = input
    .split_first()
    .ok_or(ProgramError::InvalidInstructionData)?;

    Ok(match tag {
        0 => HelloInstruction::SayHello,
        1 => HelloInstruction::SayBye,

// --snip--
```

Now when we run our client, we see that our instruction has been decoded correctly.
![console output](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7g6ffw6g5f0ctll4hi3e.png)

We get the correct operation: `Program log: Instruction Ok(SayHello)`.

Great! we're almost done. Have a go at creating the `createSayByeInstructionData` function. Which is identical to the `createSayHelloInstructionData` function except for the instruction that we need to send.

Now let's change our program to decrement the greeting counter when we received a `SayBye` operation.

## Update `process_instruction`

```diff
pub fn process_instruction(
    program_id: &Pubkey, // Public key of the account the hello world program was loaded into
    accounts: &[AccountInfo], // The account to say hello to
    instruction_data: &[u8], // Ignored, all helloworld instructions are hellos
) -> ProgramResult {
    msg!("Hello World Rust program entrypoint");
    let instruction = HelloInstruction::unpack(instruction_data)?;
    msg!("Instruction {:?}", instruction);

    // Iterating accounts is safer then indexing
    let accounts_iter = &mut accounts.iter();

    // Get the account to say hello to
    let account = next_account_info(accounts_iter)?;

    // The account must be owned by the program in order to modify its data
    if account.owner != program_id {
        msg!("Greeted account does not have the correct program id");
        return Err(ProgramError::IncorrectProgramId);
    }

    // Increment and store the number of times the account has been greeted
    let mut greeting_account = GreetingAccount::try_from_slice(&account.data.borrow())?;
+
+   match instruction {
+       HelloInstruction::SayHello => {
            greeting_account.counter += 1;
+       },
+       HelloInstruction::SayBye => {
+           greeting_account.counter -= 1;
+       },
+       _ => {}
+   }
+
    greeting_account.counter += 1;
    greeting_account.serialize(&mut &mut account.data.borrow_mut()[..])?;

    msg!("Greeted {} time(s)!", greeting_account.counter);

    Ok(())
}
```

Now that we've made a change to our program. We need to build and deploy it again.

Let's run our client.

![Console output](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mu1wct70den07565rdvi.png)

Running our client multiple times to show that our greeting count is being incremented.

Let's round this up by sending a `SayBye` operation.

```diff
// --snip--
export async function sayHello(): Promise<void> {
  console.log('Saying hello to', greetedPubkey.toBase58());
  const instruction = new TransactionInstruction({
    keys: [{pubkey: greetedPubkey, isSigner: false, isWritable: true}],
    programId,
-   data: createSayHelloInstructionData(),
+   data: createSayByeInstructionData(),
  });
  await sendAndConfirmTransaction(
    connection,
    new Transaction().add(instruction),
    [payer],
  );
}
```

```typescript
function createSayByeInstructionData(): Buffer {
  const dataLayout = BufferLayout.struct([BufferLayout.u8('instruction')]);

  const data = Buffer.alloc(dataLayout.span);
  dataLayout.encode(
    {
      instruction: 1,
    },
    data,
  );

  return data;
}
```

We run our client again.

![Console output](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/onuwgmxjzgtu1rdnpay1.png)

Our program correctly decodes the `SayBye` operation and decrements our greeting count.

Congratulations! We've managed to modify our program to specify what operation the program should perform.

[1]:https://github.com/solana-labs/example-helloworld
[2]: https://doc.rust-lang.org/std/option/enum.Option.html#method.ok_or
