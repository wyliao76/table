## Goal

This is a project extended from one of the interview questions I have encountered.
The question was to write a table of 4 seats which can be sit by players.
Need to prevent duplicate players in a table and show OOP.

For OOP we have separation of concern so table, seat, player are three classes.
Seat manages sit and un_sit and holds a player. Table holds 4 seats.

To find if a table is full,
we iterate the seat array and ask the seats if it's taken.
If all seats are taken then the table is full, return an error.

Next, to find if a player has already seated at this table,
we iterate and ask the seats if current player exists.
If exists we return duplicate player error.

Next we try to sit the current player, if taken we iterate to next, else we sit the player.

# This is a simple and easy to code example for OOP.
# However, I wanted to explore the practical design instead of just doing OOP.

First, the Seat class can be abstracted away as an array in the Table.
We can store player (pointer) or player_id in this seats array.

Second, I want a way to track if a table is full or not.
We can use an int for counting how many players or use the array size.

Finally, I need to find a way to store the status of seats, which seats are taken and which are free.
We can also consider using set for the seats to de-duplicate.

However, with my experience in C I start thinking about how would this be implemented in high level languages.
I don't like dynamic array because it would be an array with realloc or tree.
Set will probably be a HashSet.
In this case I know how many seats a table has, so I prefer a fixed size array.
HashSet sounds nice, but the overhead makes it a bad choice.
I don't want to use more cpu (hashing) and memory (sparse) for an array of 4.

# So let's use a fixed size array for player pointer.
# If we use array's index to represent the seat numbers, we can also store the seat's status.
# We check duplication by iterating through the array since it's small and spatial locality should help.
# For the table is full feature, we use an int to track how many player.
# Then we iterate to find a free seat when needed.

That's how much I can think of initially with mostly Javascript and a bit of C.
Later I wanted to do it in C and see what I can discover.

## First I did a design for small scale like a local game of 1 table. I assumed memory is a constraint.

I can use player pointers like OOP would do, and this uses 8 bytes per pointer.
So 4 * 8 = 32 bytes. We can iterate and check duplication and fullness.
Or use 1 byte to store how many players.

# So 32 + 1 = 33 bytes.

I have also considered using player_id, range from 0 to 3, each takes 1 byte to store.
So 4 bytes only, and 1 byte for player number.
However, 0 is a player_id and can't be used as null. I need something to track the status of seats.
The 1 byte for player number is uint8_t which ranges from 0 to 255, but it only tracks the player number.
If I add something else I need more memory.

### Then I discovered bitmap.

A bitmap is consecutive bits that we use to store information.
1 is set, meaning the seat is free. 0 is clear, meaning the seat is taken.
Since it uses bitwise operation for computers it would be fast.
It can be used to check if a seat is taken, if table is full and to find a free seat.

# This design use 5 bytes. (4 for player_id array and 1 for bitmap)

We can even push it to 2 bytes by storing each player_id in 2 bits.
Therefore, 1 byte for player_id array and 1 byte for bitmap.

Usually memory is not so scarce so we can use pointers and it is pretty flexible.
Bitmap is a nice addition to track player number and to find a free slot.

## So my second step is to explore a bigger game with many tables and players on one machine.

We use pointer for players and tables. Each table has a bitmap.
This should work for most time.
However, I recalled that my option instructor said if we want to scale even more,
pointers are not good because they are local to a machine.
That means if we need more computers this won't work.

## Finally, distributed system.
So we are back to array and index. Array can be mapped to a different machine.
It is also a great way to partition a fixed size of memory (Here is the players aka session_ids).
We can determine how many players and how many tables a server can have.
To keep it simple I am only gonna talk about session_id here.
An index can be used and if we align it with binary (base 2) and it works like memory cache line I have learned in school.
We can perform right swift to find which partition and it's index on the partition.
it's also possible to recover should a partition fail because array is portable.

## allocate
As I mentioned above, we make the index in a range of power of 2. Say we pick 2^3 here so we have 8 session_id.
We partition it to several servers and that's the way to scale it.
Say we want each partition to be 4 sessions and each server handles a partition.
So now we have two servers, server 0 and server 1.
2^3 right swift 2^2 (4, aka. partition size) is 2^1 = 2 (number server)

For allocate we can have a load balancer sending player to less loaded partition or whatever we like.
Then the server just incr to allocate an index (local index) and calculate the global index.

Assume the request was sent to server number 1 which has no row yet.
So server 1 add a row which has local index 0.
It calculates the global index:

global index = local index + partition size * server number
4 = 0 + 4 * 1
The global index is 4.

## lookup
For any given index we can do bitwise to find it's partition and it's local index in the partition.
Here is an example, index 4 which is 2^2.
The first partition has 0 to 3.
The second partition has 4 to 7.

Partition number = index / partition size
=> 4 / 4 = 1. (bitwise 4 >> 2)
index 4 is in partition number 1 which is correct.

global index = local index + partition size * server number
=> local index = global index - partition size * server number
=> 0 = 4 - 4 * 1
The local index is 0.

## encode/decode
We now has an index goes from 0 to 2^n - 1. (Here is player_id aka session_id).
However, we can also use this index to create something else other than integer. For example, short url like AAAAA.
If we pick 64 possible char to represent a char in the url, then we are encoding it into base64.
So an integer from our index can be encoded into a string.
Each char is a left shift in base 64 so multiply by 64 each left switch.

When 'A' is 0, AAAAA = 0*64^4 + 0*64^3 + 0*64^2 + 0*64^1 + 0*64^0 = 0

We can also customize our base64 schema, or adjust the encoding/range to generate a longer string.

## obfuscation
Surely some times we don't want to expose our index or it's base64 encoded to the client.
We could still benefit from this index though.
Let's call it internal_id and the one we show to client is public_id.
We can obfuscate it a bit by doing some math.

public_id = private_id ^ a number * a secret number

Then we encode the public_id.

short_url = encode(public_id)

## No hashing, reuse or not reuse
Here we have a distributed system for temporary session_id without hashing.
It is easy to scale by raising the power of 2.
My table example is the reuse case where our slots number is fixed and session_id is temporary.
So reclaim is needed thus the logic will be much more complicated.

There are other applications that do not reuse the index. For example, short url and payment id.
In those cases it is pretty simple, just use a big number like 2^64 which is ridiculously big to run out in our lifetime.


# template-c Repository Guide

Welcome to the `c template` repository. This guide will help you set up and run the provided scripts.

## **Table of Contents**

1. [Cloning the Repository](#cloning-the-repository)
2. [Prerequisites](#Prerequisites)
3. [Running the `generate-cmakelists.sh` Script](#running-the-generate-cmakelistssh-script)
4. [Running the `change-compiler.sh` Script](#running-the-change-compilersh-script)
5. [Running the `build.sh` Script](#running-the-buildsh-script)
5. [Running the `build-all.sh` Script](#running-the-build-allsh-script)
6. [Copy the template to start a new project](#copy-the-template-to-start-a-new-project)

## **Cloning the Repository**

Clone the repository using the following command:

```bash
git clone https://github.com/programming101dev/template-c.git
```

Navigate to the cloned directory:

```bash
cd template-c
```

Ensure the scripts are executable:

```bash
chmod +x *.sh
```

## **Prerequisites**

- to ensure you have all of the required tools installed, run:
```bash
./check-env.sh
```

If you are missing tools follow these [instructions](https://docs.google.com/document/d/1ZPqlPD1mie5iwJ2XAcNGz7WeA86dTLerFXs9sAuwCco/edit?usp=drive_link).

## **Running the generate-cmakelists.sh Script**

You will need to create the CMakeLists.txt file:

```bash
./generate-cmakelists.sh
```

## **Running the change-compiler.sh Script**

Tell CMake which compiler you want to use:

```bash
./change-compiler.sh -c <compiler>
```

To the see the list of possible compilers:

```bash
cat supported_cxx_compilers.txt
```

## **Running the build.sh Script**

To build the program run:

```bash
./build.sh
```

## **Running the build-all.sh Script**

To build the program with all compilers run:

```bash
./build-all.sh
```

## **Copy the template to start a new project**

To create a new project from the template, run:

```bash
./copy-template.sh <desitnation directory>
```

This will copy all of the files needed to start a new project.

1. Edit the files.txt
2. run ./generate-cmakelists.sh
3. run ./change-compiler.sh -c <compiler>
4. run ./build.sh

The files.txt file contains:
<executable> <source files> <header files> <libraries>

When you need to add/removes files to/from the project you must rerun the 4 steps above. 
