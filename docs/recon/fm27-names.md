# FM27: people's names

Where the game keeps the first, last, common and full names of players and staff, how it makes a new name, and how
the editor changes a player's names (#79). Read on the Windows Game Pass build 0.9.2435.0, on a save of 13 January
2028 with the Turkish leagues: 10,794 players, 9,303 members of staff.

## A person's names

A player points at his names from his object (`[player]` in the layout; a player-coach's sit 0xF8 further, as the rest
of his person block): `+0x2D8` first name, `+0x2E0` last name and `+0x2E8` common name, each an entry of one of the
game's tables of names (null for a common name he has not), and `+0x2C8` his full name, an FM string of his own (null
for none; Pedri has none, Bukayo Saka's is "Bukayo Ayoyinka Saka"). A member of staff has the same at `+0x150`,
`+0x158`, `+0x160` and `+0x140` (`[staff]`). The game shows the common name where there is one, otherwise the first
and last names.

An FM string is a `u32` length and the text in UTF-8 with a NUL, after a `u32` count of the objects that hold it.
Strings are shared: the "Saka" of the last names' table had 19 holders, the "Thomas" two of the game's entries point at
396.

## The tables

There are three tables: first names, last names and common names. An entry takes 24 bytes:

| Offset | Type | What it holds |
| --- | --- | --- |
| `+0x00` | pointer | the text, an FM string |
| `+0x08` | `u32` | its id: the save knows it by it, and it is the entry's place in the table's list by id |
| `+0x0C` | `u16` | the nation it was made for, by the game's index of nations (65535 for none): 139 for English names, 160 for Norwegian, 61 for Japanese |
| `+0x0E` | `u16` | its weight (1 for most; "Lionel" 31, the common name "Juanito" 15; 0 for a name the game made by hand) |
| `+0x10` | `u16` | 65535 for most; a number in some (1,992 to 2,385 for Korean common names) |
| `+0x12` | `u8` | flags: 0x08 in the entries the game makes for a new player; 0x00, 0x02, 0x80, 0x88, 0x84, 0x8C, 0x44 or 0x0C in the database's |
| `+0x13` | `u8` | in the entries of new players, what looks like the ethnicity of the name (0x0A Hideki Miwa, 0x03 Prince Boateng, 0x05 Jai Ganatra, 0x01 Luca Ghilarducci); 0xFF in most of the database's |
| `+0x14` | `u32` | 0 |

The weight is what the game picks names for a new person by, nation by nation: an entry the game makes for a person
it names by hand (the human manager's "(gamertag)" last name, a member of staff's) has 0, and is never picked for
anyone else.

Each table is reached two ways:

- **The list by id**, a `(begin, end, capacity)` of pointers to the entries in the order of their ids, entry `i` at
  place `i`. The game plugin keeps a pointer to it in its writable data, 0x38 (first names), 0x30 (last) and 0x28
  (common) bytes before the players' list (`[names] lists_before_players`): at `game_plugin.dll+0x53B7DA8`,
  `+0x53B7DB0` and `+0x53B7DB8` on Game Pass 0.9.2435.0, whose players' list is at `+0x53B7DE0`. The lists held
  321,748 first, 645,054 last and 96,628 common names, each with room for exactly 32 more: as many as the save held,
  and 32. The game reaches it whenever a name is known by its id.
- **The database's list**, a `(begin, end, capacity)` of pointers to the same entries in the game plugin's data
  itself, 0xD8, 0xF0 and 0x108 bytes after the players' list (beside the staff's at 0x48 and the clubs' at 0x1B8), in
  the order of their nations (65535 first): the names the game picks from for a new person of a nation. Its capacity
  was its end: no room.

The entries are the slots of one of the game's pools (`object_pool.rs`): a static at `game_plugin.dll+0x53B70D0` on
Game Pass 0.9.2435.0, of 24-byte slots, 107 chunks of 10,000, 6,530 free and 1,063,470 used. The three tables load one
after another, so the first names' last chunk ends with the first of the last names.

No other count of names was found: no `u32` 321,748 in the plugin's data, and none in the heap but in entries' ids.
The game takes the next id from the length of the list.

## The save

A save (a Game Pass save, its zstd frames joined) holds each table as a `u32` count and that many records of a `u32`
id, a `u32` length and the text: 321,748 first names with ids 0 to 321,747 in order, then 645,054 last names, then
96,628 common names. A person holds his names as their ids, `u32` each, -1 for none, and his full name as text:
"... 14 6D 02 00 00 4D CD 03 00 00 FF FF FF FF 00 14 00 00 00 Bukayo Ayoyinka Saka" for Saka (first name 158,996, last
name 249,165, no common name). The records of the tables hold no nation, weight or flags, but the save keeps them
somewhere else: entries the editor made came back from a save with the nation, weight 0 and the bytes it gave them.

## How the game makes a name

The save's tables hold names the game made as it was played: the database's first names end at id 321,703, then come
an entry with no text (321,704, nation 65535, weight 0), "Wiz" (321,705, weight 0, for a member of staff), and 42 more
first names of new players (321,706 to 321,747); the last names' table had 54 such from id 645,000 on, the human
manager's "(gamertag)" first. The game makes a new entry for nearly every new person, even for a name it has: two
players got their own "Thomas" entries (ids 321,721 and 321,726), pointing at the same string, with the player's
nation, weight 1, flags 0x08 and his ethnicity. Brothers share an entry: Lorenzo and Alessandro Granelli point at the
same "Granelli".

## Changing a player's names

The editor names a player as the game names a new person (see `crates/fmlibre-core/src/names.rs`):

1. It takes the first free slot of the pool of entries, as the game takes one.
2. It writes the entry there: its text, a string in memory of the editor's own laid out as the game's are, with a
   count of holders that never runs out; the next id, the length of the list by id; the nation and the other bytes of
   the entry it replaces (for a common name he had none of, his last name's); and weight 0, so that the game never
   picks the name for anyone else.
3. It puts the entry at the end of the list by id, within the room the list has.
4. It points the player at it.

The database's list, which the game picks names for new people from, is left as it is: it has no room, and the name is
the player's alone. A full name is a string of the editor's own the player is pointed at. A common or full name is taken
away by nulling the pointer.

Checked on Game Pass 0.9.2435.0 with `fmlibre-probe rename`: Pedri renamed and put back (`--restore`), then Erling
Haaland given "Erlingo Testland" with the full name "Erlingo Test Testland", and Bukayo Saka the common name "Saka
Test". The lists by id took the new entries at their ends (ids 321,749, 645,055 and 96,628), and `fmlibre-probe names`
read them at the places their ids say.

Then in the game, with the app's Rename dialog: Haaland was given back "Erling", "Haaland" and "Erling Braut Haaland" (new
entries of weight 0, ids 321,752 and 645,058), and Saka the common name "Bukayo Saka" (96,631); the game was saved and
the save loaded. The tables were built again from the save, in other slots of the pool, and held every entry the editor
had made at its id, with its nation, weight 0 and bytes: the used ones with Haaland and Saka pointing at them, the
unused ones (the tests') with no one. The game then played two days and named its new players with the next ids
(321,753 "Celso Fernando", 645,059 "Jung"), after the editor's. No crash.

An undo points the player at his old entry or string again. The new entry stays in its table, used by no one, as some
of the game's own are, and is saved with it: giving its slot back to the pool could leave the game holding a pointer
to a slot it then hands out for another name.

When a list by id has no room left, the editor refuses a new name of that kind (`names.noRoom`): the game gives the list
more room when it makes names itself (a youth intake, new staff), and the save loaded again has 32 more.

## Not known yet

- What `+0x10` and the flags at `+0x12` mean.
- The Mac build: its layout places no tables of names yet, so the app shows names there without changing them.
