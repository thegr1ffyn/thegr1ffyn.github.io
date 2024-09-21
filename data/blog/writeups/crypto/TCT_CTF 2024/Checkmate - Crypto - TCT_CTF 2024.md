---
title: Checkmate - Crypto - TCT_CTF
date: 2024-09-21
tags:
  - ctf
  - capture-the-flag
  - crypto
  - chessencryption
  - stego
draft: false
---
### Description
No human would play like this lol
### File
[Download here](https://github.com/thegr1ffyn/thegr1ffyn.github.io/blob/main/public/static/writeups/output.pgn)
### Solution
Starting with the file, we can simply check this file type on Google where we get the following answer.
`Portable Game Notation (PGN) is a standard plain text format for recording chess games (both the moves and related data), which can be read by humans and is also supported by most chess software.`

After some research, I found out that it is possible to exploit `.pgn` files to hide data inside.
Some of the resources, I went through are as follows.
 - [Chestega chess-steganography](https://www.semanticscholar.org/paper/Chestega%3A-chess-steganography-methodology-Desoky-Younis/a6ba82be2139308ccdd5bcc128b4f8db10b87b3c)
 - https://www.giac.org/paper/gsec/4127/stenganography-chess-pgn-standard-format/106529
 - https://lobste.rs/s/ikxxhw/hiding_messages_chess_games
 - https://github.com/Alheimsins/chess-steg-cli
 - https://incoherency.co.uk/chess-steg/
 - https://indianexpress.com/article/puzzles-and-games/info/chess-games-cloud-storage-creative-coding-cryptography-brainteasers-9567020/
 - https://www.reddit.com/r/chess/comments/ad9nl2/chess_steganography_encodedecode_data_in_chess/
 

However, after trying all the approaches, it was a good learning point but not really helpful with the exploitation.

I found https://github.com/Alheimsins/chess-steg-cli which is a tool that unstegs pgn moves. Since, we had thousands of moves, automation is my best friend for such tasks.
I wrote two scripts, where I first sorted the given `output.pgn` file to `moves` and then read those moves to check for any unsteg moves.
The scripts were as follows:
```python
def read_and_write_pgn(file_path, output_file_path):
    with open(file_path, 'r') as file:
        lines = file.readlines()

    moves = set()  # Use a set to ensure uniqueness
    move_started = False

    for line in lines:
        line = line.strip()
        if line.startswith('1.'):
            move_started = True
            moves.add(line)  # Add the first move line
        elif move_started:
            if line == '*':
                break  # Stop when reaching the end of moves
            if line:  # If the line is not empty
                moves.add(line)

    # Write the unique moves to a new file
    with open(output_file_path, 'w') as output_file:
        for move in sorted(moves):
            output_file.write(move + '\n')

# Usage
input_file_path = 'output.pgn'  # Update this path
output_file_path = 'newpgn.txt'  # Specify output file path
read_and_write_pgn(input_file_path, output_file_path)
```
and used the following to check
```bash
#!/bin/bash

# Path to your output file with unique moves
input_file="newpgn.txt"

# Loop through each line in the file
while IFS= read -r line; do
    # Run the chess-steg command with the current line
    result=$(chess-steg -u "$line" 2>&1)  # Capture both stdout and stderr

    # Check for an error in the output
    if [[ $result == *"Error"* ]]; then
        echo "Output for move : Not found"
    else
        echo "Output for move '$line':"
        echo "$result"
    fi
    echo "-----------------------------"
done < "$input_file"
```
But, it wasnt helpful at all. Wasted half of my time here, but this is another approach to a future CTF challenge.

After some more recce, I found a video which I think is a beautiful explanation.
https://www.youtube.com/watch?v=TUtafoC4-7k&t=173s

I found a tool named `ChessEncryption` which encrypt files into large sets of Chess games stored in PGN format. This is a library so you will need to import functions from `decode.py` and `encode.py` to use this
https://github.com/WintrCat/chessencryption

However, I was unable to get this tool/library setup on my system due to lack of documentation. (please make good documents guys).

I had to get my hands dirty and I edited the script according to my need where my final script became
```python
from time import time
from math import log2
from chess import pgn, Board
from io import StringIO

def read_pgn_file(pgn_file_path: str) -> str:
    with open(pgn_file_path, "r") as file:
        return file.read()

def get_pgn_games(pgn_string: str) -> list[pgn.Game]:
    games = []
    pgn_reader = StringIO(pgn_string)

    while True:
        game = pgn.read_game(pgn_reader)
        if game is None:
            break
        games.append(game)

    return games

def decode(pgn_string: str, output_file_path: str):
    start_time = time()

    total_move_count = 0

    games = get_pgn_games(pgn_string)

    with open(output_file_path, "w") as output_file:
        output_file.write("")

    output_file = open(output_file_path, "ab")
    output_data = ""

    for game_index, game in enumerate(games):
        chess_board = Board()

        game_moves = list(game.mainline_moves())
        total_move_count += len(game_moves)

        for move_index, move in enumerate(game_moves):
            legal_move_ucis = [
                legal_move.uci()
                for legal_move in list(chess_board.generate_legal_moves())
            ]

            move_binary = bin(
                legal_move_ucis.index(move.uci())
            )[2:]

            if (
                game_index == len(games) - 1 
                and move_index == len(game_moves) - 1
            ):
                max_binary_length = min(
                    int(log2(
                        len(legal_move_ucis)
                    )),
                    8 - (len(output_data) % 8)
                )
            else:
                max_binary_length = int(log2(
                    len(legal_move_ucis)
                ))

            required_padding = max(0, max_binary_length - len(move_binary))
            move_binary = ("0" * required_padding) + move_binary

            chess_board.push_uci(move.uci())

            output_data += move_binary

            if len(output_data) % 8 == 0:
                output_file.write(
                    bytes([
                        int(output_data[i * 8 : i * 8 + 8], 2)
                        for i in range(len(output_data) // 8)
                    ])
                )
                output_data = ""

    print(
        "\nSuccessfully decoded PGN with "
        + f"{len(games)} game(s), {total_move_count} total move(s)"
        + f" ({round(time() - start_time, 3)}s)."
    )


pgn_file_path = "output.pgn"
output_file_path = "output.bin"

pgn_string = read_pgn_file(pgn_file_path)
decode(pgn_string, output_file_path)
```
After running the script, it took around 90 seconds and I was happy to get the following response.
```bash
└─$ python3 new.py                                   
Successfully decoded PGN with 2811 game(s), 586350 total move(s) (94.426s).
```
I checked the output, and we had the PNG header which means we got an image. Voila :)
```bash
└─$ cat output.bin           
�PNG
▒
IHDRx-�E9PLTEabb
```
Moving this data to a `png` file, and opening the image, we had our flag :)
```bash
mv output.bin output.png
```

`TCT{CH355_1S_G00D_F0R_M1ND_A5_W3LL_A5_3NCRYP710N}`

Appreciation for the challenge author [Strangek](https://github.com/kumailzaidi23) for a very unique challenge

### Code In-depth explaination
This code reads a Portable Game Notation (PGN) file containing chess games, decodes the games, and then converts the moves into a binary format that is saved in a file. Here’s a breakdown of what each part of the code does:
### 1. **Import Statements**
   - `from time import time`: This imports the `time` function to measure the execution time of the program.
   - `from math import log2`: This imports the `log2` function to compute the binary logarithm (log base 2).
   - `from chess import pgn, Board`: Imports the `pgn` module to read PGN files and the `Board` class to simulate chess positions.
   - `from io import StringIO`: Imports `StringIO` to treat strings as file-like objects.
### 2. **Functions**

#### a) `read_pgn_file(pgn_file_path: str) -> str`
   - Reads the contents of a PGN file located at `pgn_file_path` and returns the content as a string.

```python
def read_pgn_file(pgn_file_path: str) -> str:
    with open(pgn_file_path, "r") as file:
        return file.read()
```

#### b) `get_pgn_games(pgn_string: str) -> list[pgn.Game]`
   - Takes a PGN string and returns a list of `pgn.Game` objects by reading and parsing the PGN data.
   - Uses `StringIO` to treat the PGN string as a stream and reads each game until no more games are found.

```python
def get_pgn_games(pgn_string: str) -> list[pgn.Game]:
    games = []
    pgn_reader = StringIO(pgn_string)

    while True:
        game = pgn.read_game(pgn_reader)
        if game is None:
            break
        games.append(game)

    return games
```

#### c) `decode(pgn_string: str, output_file_path: str)`
   - This is the main function that processes the chess games and writes the moves in binary format to the `output_file_path`.
   
   Key steps inside the function:
   
   - **Time Measurement**: It records the start time using `start_time = time()`.
   
   - **Initialize Move Counting**: `total_move_count` keeps track of the total number of moves in all games.
   
   - **Fetch Games**: Calls `get_pgn_games` to parse the PGN string and returns a list of games.
   
   - **Prepare for Output**: Opens the output file in binary mode (`"ab"`) for writing the moves in binary form.

   - **Loop Over Games**: For each game, it sets up a `Board` object representing the chessboard, simulates the moves, and processes each move.
   
     - For each move, it generates all legal moves (`chess_board.generate_legal_moves()`), finds the index of the actual move in the list of legal moves, and encodes that index as a binary string.
     
     - It determines the required number of bits to represent the move using `log2`, ensuring proper padding to fit the binary format.
   
     - Once enough bits are collected to form a byte (8 bits), the function writes them as bytes to the output file.

   - **Edge Case Handling**: For the last move of the last game, it ensures that the binary data fits correctly into the 8-bit format, truncating or padding as needed.

```python
def decode(pgn_string: str, output_file_path: str):
    start_time = time()

    total_move_count = 0
    games = get_pgn_games(pgn_string)

    with open(output_file_path, "w") as output_file:
        output_file.write("")

    output_file = open(output_file_path, "ab")
    output_data = ""

    for game_index, game in enumerate(games):
        chess_board = Board()
        game_moves = list(game.mainline_moves())
        total_move_count += len(game_moves)

        for move_index, move in enumerate(game_moves):
            legal_move_ucis = [
                legal_move.uci()
                for legal_move in list(chess_board.generate_legal_moves())
            ]

            move_binary = bin(
                legal_move_ucis.index(move.uci())
            )[2:]

            if (
                game_index == len(games) - 1 
                and move_index == len(game_moves) - 1
            ):
                max_binary_length = min(
                    int(log2(
                        len(legal_move_ucis)
                    )),
                    8 - (len(output_data) % 8)
                )
            else:
                max_binary_length = int(log2(
                    len(legal_move_ucis)
                ))

            required_padding = max(0, max_binary_length - len(move_binary))
            move_binary = ("0" * required_padding) + move_binary

            chess_board.push_uci(move.uci())

            output_data += move_binary

            if len(output_data) % 8 == 0:
                output_file.write(
                    bytes([
                        int(output_data[i * 8 : i * 8 + 8], 2)
                        for i in range(len(output_data) // 8)
                    ])
                )
                output_data = ""

    print(
        "\nSuccessfully decoded PGN with "
        + f"{len(games)} game(s), {total_move_count} total move(s)"
        + f" ({round(time() - start_time, 3)}s)."
    )
```

### 3. **Main Program Flow**
   - Reads the PGN data from `output.pgn`.
   - Calls the `decode` function, which decodes the chess games and writes the moves in binary to `output.bin`.
   
```python
pgn_file_path = "output.pgn"
output_file_path = "output.bin"

pgn_string = read_pgn_file(pgn_file_path)
decode(pgn_string, output_file_path)
```

### Key Points:
- **Binary Move Representation**: Each move is encoded as a binary index based on its position in the list of legal moves. This allows compression of the chess game data.
- **Efficient Writing**: Data is written to the binary output file only when a full byte (8 bits) of move data is available.
- **Logging**: The program prints a summary of how many games and moves were processed, along with the time taken to decode.