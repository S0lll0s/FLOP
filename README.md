FLOP
====

The FLOP filesystem is designed for the use with the Mackapar 3.5" Floppy Drive (M35FD).


[Specification:](https://gist.github.com/S0lll0s/5446051#file-flop-specification-draft-1)

Introduction
============
 
The FLOP filesystem is designed for the use with the Mackapar 3.5" Floppy Drive (M35FD).
The M35FD consists of 1440 sectors of 512 words each (737,280 words total).
 
The first seven sectors are the main header, followed by the file list.
One block is a sector. It just stores pure data.
Blocks are allocated back-to-front, to reduce collision between the file list and the data blocks.
 
 
Header
------
 +----------+------------------------------------+
 |   size   |   description                      |
 +----------+------------------------------------+
 |   1 w    | FLOP magic number (0x8adf)         |  ---\
 |          |                                    |     |
 |   1 w    | version                            |     |
 |          |                                    |     |
 |   1 w    | header size                        |     |- block 0
 |          |                                    |     |
 |   16 w   | drive name (0-padd, packed string) |     |
 |          |                                    |     |
 |   1 w    | filelist size (words)              |     |
 |          |                                    |     |
 |  492 w   | reserved for more header data      |  ---/
 |-----------------------------------------------|
 |  512 w   | block list, 512 * 6 = 3072         |  ---\
 |  512 w   |             we have 2 words        |     | - blocks 1 - 6
 |  512 w   |             per block              |  ---/
 |-----------------------------------------------|
 |  512 w   | file list (alphabetic order)       |  ---\
 |  512 w   . file list (alphabetic order)       |     .
 |  512 w   . file list (alphabetic order)       |     .- blocks 4 .. n
 |  512 w   | file list (alphabetic order)       |  ---/
 |__________|____________________________________|
  
 
Block list entry
----------------
 +------+----------------------------------+
 | size |   description                    |
 +------+----------------------------------+
 |  1w  | H              L                 |
 |      | lllllllllfffffff                 |
 |      | \___ _/___ ____/                 |
 |      |     v     v                      |
 |      |     |     +-----  last set word  |
 |      |     +-----------  unused flags   |
 |      |                                  |
 |  1w  | H              L                 |
 |      | ttttFFFFFFFFFFff                 |
 |      | \__/____ ____/_/                 |
 |      |  v      v     |                  |
 |      |  |      |     +-  unused flags   |
 |      |  |      +-------  file id        |
 |      |  +-- type:                       |
 |      |    0: unused; 1: header;         |
 |      |    2: blocklist, 3: filelist     |
 |      |    4: data                       |
 |______|__________________________________|
 
 
File list entry
---------------
 +----------+----------------------------------+
 |   size   |   description                    |
 +----------+----------------------------------+
 | max 16 w | filename (0-term, packed string) |
 |          |                                  |
 |   1 w    | file size (words)                |
 |          |                                  |
 |   1 w    | # of allocated blocks            |
 |          |                                  |
 |   n w    | block list:                      |
 |          |  H              L                |
 |          |  00000fBBBBBBBBBB                |
 |          |  \_ _//____ ____/                |
 |          |    v  |    v                     |
 |          |    |  |    +------- block id     |
 |          |    |  +-- [is block filled?]     | Better not implement that, really sucks for usability
 |          |    +-- unused                    |
 |__________|__________________________________|
 
 
Untested code snippets
----------------------
 getting info about a block X (ex. 258):
  
  SET B, X     ; B, X = 258
  MOD B, 256   ; B = 2
  SUB X, B     ; X = 256
  DIV X, 256   ; X = 1
  SUB B, 1     ; B = 1
  MUL B, 2     ; B = 2
 
  SET A, 2     ; load sector X to [Y, Y+512]
  SET Y, [temp]
  HWI [m35fd_id]
 
  ADD Y, B     ; Y = temp + 2
  ; Y now points to the first word of the block info
 
 checking if a block is free (Y points to the corresponding block info):
  SET B, [Y+1] ; read 2nd word
  SHR B, 12    ; shift to type only
  IFE B, 0
     JSR sth   ; block isn't used