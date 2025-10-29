IQDB — On-Chain Table Writer & Reader (README)

This repository demonstrates how to use the IQ Database program on Solana to:
- Initialize a Root & TxRef PDA
- Create tables with column names
- Insert rows into those tables (as JSON)
- Read & pretty-print recent rows per table by scanning the TxRef PDA

The project provides two scripts:
- writer (src/iqdb.ts): builds & sends instructions (initialize root, create table, write rows)
- reader (src/reader.ts + src/reader_core.ts): scans recent TxRef transactions and prints per-table rows

====================================================================
0) Prerequisites
====================================================================
- Node.js ≥ 18
- ts-node / typescript
- @coral-xyz/anchor
- @solana/web3.js
- A deployed program + IDL (PROGRAM_ID and target/idl/iq_database.json)
- solana-cli configured with your wallet:
    solana config set --url https://api.devnet.solana.com
    solana config set --keypair ~/.config/solana/id.json

====================================================================
1) Configuration
====================================================================
File: src/configs.ts
---------------------------------
export const configs = {
  network: process.env.NETWORK_URL || "https://api.devnet.solana.com",
  programId: process.env.PROGRAM_ID || "7Vk5JJDxUBAaaAkpYQpWYCZNz4SVPm3mJFSxrBzTQuAX",
  rootSeed: "iqdb-root",
  tableSeed: "iqdb-table",
  txRefSeed: "iqdb-txref",
  instructionSeed: "instruction",
  targetSeed: "target",
};

Ensure PDA seed usage in src/provider/pda.provider.ts exactly matches on-chain seeds.

====================================================================
2) Install & Run
====================================================================
pnpm install     # or npm install
# Writer example
npx ts-node src/iqdb.ts
# Reader example (uses current solana wallet automatically)
npx ts-node src/reader.ts

====================================================================
3) Writer — create table and insert rows
====================================================================
Key functions (src/iqdb.ts):
- getProvider(): AnchorProvider (uses AnchorProvider.local())
- getProgram(): Program loaded from local IDL
- txSend(connection, tx, keypair): send+confirm a built Transaction
- initializeRoot(signerPubkey): returns a Transaction (initialize root + txref PDAs)
- createTable(signer, root, tableName, columnNames): returns a Transaction
- writeRow(signer, root, table, txRef, tableName, rowJson): returns a Transaction

Example usage inside main():
------------------------------------------------------------------
const provider = getProvider();
const { connection } = provider;
const signer = provider.wallet;
const keypair = (signer as any).payer;

const rootPda = pdaRoot(signer.publicKey);

// 1) Initialize root+txrefs
const initTx = await initializeRoot(signer.publicKey);
const initSig = await txSend(connection, initTx, keypair);
console.log("init tx:", initSig);

// 2) Create table
const crtTx = await createTable(signer.publicKey, rootPda, "youtube", ["name", "session_pda"]);
const crtSig = await txSend(connection, crtTx, keypair);
console.log("create_table tx:", crtSig);

// 3) Insert a row
const txRef = pdaTxRef(signer.publicKey);
const table = pdaTable(rootPda, new TextEncoder().encode("youtube"));
const wrTx = await writeRow(signer.publicKey, rootPda, table, txRef, "youtube", "{ name:'cat_meme', session_pda:'fkdjskfjdlkjflkdjksakfjdaf' }");
const wrSig = await txSend(connection, wrTx, keypair);
console.log("write_row tx:", wrSig);

Notes:
- We intentionally use .instruction() + sendAndConfirmTransaction instead of .rpc()
  to avoid "Provider.sendAndConfirm must be implemented" errors.
- All on-chain string/Vec<u8> args must be Buffer (e.g., Buffer.from("name","utf8")).

====================================================================
4) Reader — pretty print rows per table
====================================================================
Files: src/reader.ts, src/reader_core.ts

- BorshAccountsCoder / BorshInstructionCoder decode accounts & instructions from IDL.
- Reads Root -> table metadata (columns) and TxRef PDA recent signatures.
- Decodes transactions and collects rows per table for:
  - write_data(table_name, row_json_tx)
  - database_instruction(table_name, mode, target_tx, content_json_tx)
- Pretty prints in the requested style.

Programmatic usage:
------------------------------------------------------------------
await readTxRefPretty({
  idlPath: "target/idl/iq_database.json",   // Set to your IDL path
  userPubkey,                               // Uses current solana wallet by default
  maxTx: 100,                               // how many recent tx to scan
  perTableLimit: 20                         // optional, rows per table printed
});

Sample output:
------------------------------------------------------------------
catfood { columns: ['name','price'] }
L rows
  { name:'cat_meme', price:100 }
  { name:'another_cat', price:250 }

coffee { columns: ['name','price'] }
L rows
  { name:'americano', price:400 }

====================================================================
5) Troubleshooting
====================================================================
A) expected environment variable ANCHOR_WALLET is not set
- AnchorProvider.local() requires a local keypair.
  Fix:
    solana config set --keypair ~/.config/solana/id.json

B) This function requires 'Provider.sendAndConfirm' to be implemented.
- Use .instruction() and sendAndConfirmTransaction() (already implemented in writer).

C) Blob.encode[data] requires Buffer as src
- Pass Buffer for string/bytes Vec<u8> args:
    const nameBuf = Buffer.from("catfood","utf8");
    const colsBuf = ["name","price"].map(s => Buffer.from(s,"utf8"));

D) No tables printed / tableNames: []
- Ensure your PROGRAM_ID and seeds match on-chain program.
- Ensure you created the table and wrote rows from the same wallet you're reading.
- Ensure reader idlPath points at the correct IDL used to decode transactions.

====================================================================
6) Project Structure
====================================================================
src/
  iqdb.ts            # writer (initializeRoot/createTable/writeRow + txSend)
  reader.ts          # entry script for reading using current wallet
  reader_core.ts     # core logic for decoding & pretty printing
  provider/
    pda.provider.ts  # PDA helpers (must match on-chain seeds exactly)
  configs.ts         # network + programId + seeds
target/
  idl/
    iq_database.json # program IDL

