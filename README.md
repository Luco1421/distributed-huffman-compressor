# Distributed Huffman Compressor

A Huffman file compressor distributed across multiple machines over TCP
sockets. A central coordinator splits an input file into N byte-range
chunks and sends one to each connected worker; each worker computes a local
byte-frequency table over its chunk and sends it back. The coordinator
merges every partial table into one global frequency table, builds a single
Huffman tree/code table from it, and broadcasts that table to every worker.
Each worker then Huffman-encodes its own chunk locally and returns the
compressed bytes; the coordinator concatenates all chunks (re-packing each
chunk's individual bit-padding) into one final compressed file, plus a
separate code-table file.

Decompression is a separate, single-machine step: it rebuilds the Huffman
tree from the saved code table and decodes the compressed file bit by bit.

## How "distributed" works here

- **Coordinator** (`code/Compressor/Server/Central`): a TCP server on port
  `8585` that accepts N worker connections, then splits and streams a file
  chunk to each worker (one pthread per worker), later broadcasts the
  merged code table and collects each worker's compressed chunk back.
- **Worker** (`code/Compressor/Server/Workers`): connects out to the
  coordinator, receives its chunk, computes a local frequency table,
  Huffman-encodes its chunk once it receives the global code table, and
  sends the result back. Built and tested across separate machines using an
  ngrok TCP tunnel between coordinator and workers.
- **Decompressor** (`code/Decompressor`): standalone, local, single-threaded
  — rebuilds the tree from the saved table and walks the compressed file.

## Stack

C (POSIX), raw BSD sockets (`sys/socket.h`, `netinet/in.h`) for networking,
pthreads for concurrency on the coordinator side. No external dependencies,
no framework — plain per-component Makefiles.

## Building and running

```bash
# Coordinator
cd code/Compressor/Server/Central
make
./server
# prompts for the number of workers, then waits for that many TCP connections

# Worker (run once per machine/connection)
cd code/Compressor/Server/Workers
make
./a.out

# Decompressor (local, after a compression run finishes)
cd code/Decompressor
make
./decompress
```

Edit `code/config.h` first — it hardcodes the input file path, output paths
for the compressed file/code table, and the coordinator's address that
workers connect to.

There's also a standalone Huffman-tree demo, unrelated to the network
pipeline, in `code/Compressor/Huffman/` (`make && ./test < test_input`).

## About this fork

This is a fork of a team project built for a Data Structures course, kept
here to document the networked coordinator/worker design and my
contribution to it. Original team: Alejandro Cerdas, Danilo Duque, Kener
Castillo, and Pablo Pérez ([Luco1421](https://github.com/Luco1421)) — see
[the original repository](https://github.com/DaniloDuque/distributed-huffman-compressor)
for the full commit history.

## License

MIT — see [LICENSE](LICENSE).
