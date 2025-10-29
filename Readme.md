# LINKS 
IQLABS database SDK 
https://github.com/IQCoreTeam/IQDBSDK

Example of using SDK
https://github.com/IQCoreTeam/IQDBSDKTEST

Contract
https://github.com/IQCoreTeam/IQ_database_contract

Database Website
https://github.com/IQCoreTeam/IQDBWebsite

# About IQDB
IQ DB by @Zo and @Spacebun

How did we achieve O(1) lookups?

Here’s how:

We structured the Root Table like this:
```
pub struct Root {
    pub creator: Pubkey,
    pub table_seeds: Vec<Vec<u8>>, // [cat_food, zozo, IQTeamcall]
    pub global_table_seeds: Vec<Vec<u8>>, // includes all ext tables
}
```
It stores the owner of the tables and the seed info used to find other tables.
Each seed is created by hashing the table name — that way, even if the name is long (like in relational structures), it can still form a valid PDA.

We separate the seeds into two lists:
    •    Local seeds: only for regular tables
    •    Global seeds: includes every ext table

So, when reading from the root, we first fetch the table info and then use those seeds to locate the tables.

Each table looks like this:
```
#[account]
pub struct Table {
    pub column_names: Vec<Vec<u8>>,
    pub id_col: Vec<u8>,
    pub ext_keys: Vec<Vec<u8>>,
    pub name: Vec<u8>,
}
```
It includes column info, the column used as an ID, any external keys, and for UX, the unhashed table name.

Here’s the key idea —
All structures use PDAs, but the actual data comes from IQ Code-In transactions.
Each on-chain inscription (transaction ID or session PDA) acts as a data record.
By sending these transactions to the table’s PDA, we can write unlimited data into that table.

So, when reading a table:
    1.    We start from the Root Table and get the seed info.
    2.    We fetch the PDA using the desired seed — this gives us the table’s transactions (data) and its settings.
    3.    After reading all transactions from that PDA, we fetch another PDA with the same seed + "instruction" to get the update/delete logs.
    •    Each log includes a target transaction and new data info.
    •    If data exists, it’s an update; if it’s empty, it’s treated as a delete.
