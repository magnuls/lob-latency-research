# Latency Research on Different Order Book Data Structures

- Fed Nasdaq ITCH feed and measured per update with the cycle counter.

![AAPL latency distribution by order table](results/m3_e2e_real.png)

## Overview

Inspired by [David Gross CppCon talk](https://www.youtube.com/watch?v=sX2nF1fW7kI). Milestones 1 (the benchmark harness), 2 (three ways to store price levels), and 3 (the order id hash table) are finished and measured.

- Feeds
  - `ob_genfeed` writes a synthetic and customizable feed for benchmarking
  - `ob_itch` converts a Nasdaq ITCH 5.0 day file into one feed per symbol
  - `ob_validate` checks that feed makes sense before it is used for benchmarking
- Books (`ob/`). Each one is a template on which order table it uses, so any level structure can be paired with any table
  - `ReferenceBook`: a slow but correct book used by ob_diff to check against our fast books
  - `MapBook`: `std::map` map<price, volume> for bids and asks and a
     hash table of orders. Each order holds an iterator to its price level, modifying or deleting O(1) on average
  - `VectorBook`: sorted vector of price levels with best price at front of vector, found with lower_bound. Inserting a new level near the top has to shift every level behind it
  - `ReverseVectorBook`: same as VectorBook but best price is located at the back, so inserting near the top only shifts a few levels. The search algo used for finding a price level is a template parameter (lower_bound, branchless, linear)
  - `TableOnlyBook<Table>`: a hash map from order ID to a record holding side, price, and volume. Table is the hash map implementation (StdOrderTable, FlatIdentityTable, or FlatMixTable). Add inserts the record, modify updates its volume, and delete removes it. It keeps no per-price volume totals. Its benchmarked time minus StubBook’s is the cost of the order-ID operations alone.
  - `StubBook`: an empty book that increments a counter on every update. It measures the cost of everything around the book, reading and decoding each message, calling the book, and timing it.
 

- Order tables (`ob/OrderTable.hpp`), picked with `--table`. A modify or delete arrives with only an order id, so every book needs a table from id to the order's -> {side, price, volume} and pays for it on two of the three message types

 - `std`: `std::unordered_map`, the baseline. Allocates a node for every add and chases a pointer on every lookup, so it is slow and we measure everything else against it

- `identity` and `mix`: our own flat open-addressing table. One array of {id, record} slots, power of two size so the modulo is a mask, linear searching so a collision walks to the next slot, usually in the same or the next cache line, backward-shift deletion so deletes don't leave any marker behind. It is sized once at startup to the next power of two at or above twice the feed's peak live order count, so load stays at or below  0.5 (ratio of filled slots to total slots), so it never grows during a run, and it makes zero heap allocations while running. Going past half load throws a std::length_error, which main catches, prints, and returns 1, ending the run.

  - `identity` uses the low bits of the ID as the slot number. This costs nothing, but if many IDs share  low bits, they pile into the same area and searches get long.
  - `mix` first multiplies the ID by a fixed 64-bit constant and XORs the top half into the bottom half. This costs a few cycles but spreads any ID pattern evenly.
 

- Benchmark (`ob_bench`)
  - times blocks of updates with the CPU cycle counter, drops first ~1% of samples, counts total heap allocation calls
    - We drop the first ~1% since noise such as cold caches, first-touch page faults, and a low CPU clock speed can affect our benchmark.
  - Pre-grow heap to avoid page faults, optionally scramble which blocks you allocate, ASLR turned off (memory layout is the same every run), run five times, and if the difference between medians > 5% we flag the run as invalid
  - `--id-stride N` multiplies every order's id by N before replay. With N equal to the table size, every id lands on the same slot, the worst case for the identity hash
  - `--prefetch` prefetches the next update's table slot while the current one is being processed. Only possible because the whole feed is known in advance, so it is not a live feed technique
  - After each run, it prints how full the table got and how far the average and the worst lookup had to walk from the slot its ID hashes to, so we can see if the half-load sizing is actually working
- Correctness
  - `ob_diff` feeds the same data feed into the ReferenceBook and a fast book at the same time, compares the two after every message, and stops at any difference. Runs every book on every table
  - `ob_tests` unit tests for every book and table, the ITCH parser, the feed validator, and ob_diff. Includes one broken book to make sure the test works. Passes in Release, Debug, and ASAN+UBSAN builds
- Results. Real AAPL order flow from 2019-12-30, 1.6M updates, on an Apple M4 Max with clang 17 at `-O2`. Every number is the median of five separate launches, in ns per update

  Milestone 2 compared the ways to store price levels, all using `std::unordered_map` for the order table:

  | book | median ns | allocations per add |
  | --- | --- | --- |
  | ref | 81.8 | 2.20 |
  | map | 40.5 | 1.21 |
  | map, randomized heap | 46.1 | 1.21 |
  | forward vector | 106.1 | 0.99 |
  | reverse vector, lower_bound | 51.6 | 0.99 |
  | reverse vector, branchless | 50.8 | 0.99 |
  | reverse vector, linear | 72.9 | 0.99 |

  Milestone 3 swapped the order table under the best books. These were measured on battery with the machine busy, so they are provisional until re-run on a quiet machine. The differences are many times bigger than the run-to-run noise, so the ranking stands:

  | book | std::unordered_map | flat, identity hash | flat, mix hash | allocations per add, std to flat |
  | --- | --- | --- | --- | --- |
  | table alone | 20.4 | 10.0 | 9.8 | 0.99 to 0 |
  | reverse vector, branchless | 45.9 | 31.7 | 32.1 | 0.99 to 0 |
  | reverse vector, linear | 66.7 | 52.4 | 52.6 | 0.99 to 0 |
  | map | 47.9 | 26.4 | 26.2 | 1.21 to 0.23 |

  ![Order table alone](results/m3_lookup.png)

  ![Map with sequential versus randomized allocation](results/m2_map_seq_vs_rand.png)

  ![Median against new level rate](results/m2_sweep.png)

- What the numbers say
  - The order table was the slowest part of every update. Swapping `std::unordered_map` for the flat table takes about 14 ns off every book and brings the vector books to zero allocations.
  - The identity hash is just as fast as the mix hash on a typical update but slows on a synthetic feed on sequential orders. On the synthetic feed with 1M live orders and sequential ids, the ids wrap around the table and pile up, so the average lookup walks 102 slots, and the slowest 1% take over 2 µs (microseconds = 1,000 ns), while the mix hash walks 0.6 slots and stays at 50 ns. Multiply the ids by the table size, and the identity hash takes 14 µs per lookup. The mix hash costs nothing measurable, so we find it's best to use
  - Prefetching the next update's slot made no difference on this machine. The whole table fits in L2 cache, so the slot is already there
  - Putting the best price at the back of the vector halves the forward vector's cost, because real books are thousands of levels deep and new levels usually appear at the top. Linear search loses on this data because the level being touched is usually not within the first few price levels


## Build and run

Requirements: CMake 3.20+, a C++23 (results were measured with clang 17), zlib headers, `curl`, and Python 3 with matplotlib for the chart scripts. `fetch_itch.sh` checks the download with `md5`, so it only runs on macOS as written.

If you have no requirements...

```sh
# macOS: Xcode command line tools give clang, zlib, curl, and md5; Homebrew gives CMake
xcode-select --install
brew install cmake
python3 -m pip install matplotlib

# Ubuntu 24.04 / Debian: GCC 13 or newer is needed for C++23
sudo apt update
sudo apt install build-essential cmake zlib1g-dev curl python3 python3-matplotlib
```
If you have the requirements...
```sh
git clone https://github.com/magnuls/lob-latency-research && cd lob-latency-research

# build (Release by default) and run the tests
cmake -S . -B build && cmake --build build
ctest --test-dir build

# tests again under AddressSanitizer and UndefinedBehaviorSanitizer
cmake -S . -B build-asan -DOB_ASAN=ON -DCMAKE_BUILD_TYPE=Debug && cmake --build build-asan
ctest --test-dir build-asan

# real data: downloads one Nasdaq trading day (about 3.5 GB) into data/
./scripts/fetch_itch.sh
./build/ob_itch --in data/12302019.NASDAQ_ITCH50.gz --symbols AAPL --out-dir results

# synthetic data: the seed 1 feed used in the results (50M updates)
./build/ob_genfeed --out results/feed_s1.bin --seed 1 --count 50000000

# single runs
./build/ob_bench --feed results/feed_itch_AAPL.bin --book reverse_branchless --table mix --hist results/hist.csv
./build/ob_bench --feed results/feed_itch_AAPL.bin --book table --table identity --id-stride 65536   # every ID lands on the same slot, the worst case for the identity hash

# check every book on every table against the reference book
./build/ob_diff --feed results/feed_itch_AAPL.bin --book all --table all

# full result tables and charts (need both feeds above)
./scripts/bench_all.sh                                    # milestone 2: every price-level structure
./scripts/bench_m3.sh && ./scripts/probe_sweep.sh         # milestone 3: every book on every table, and slots checked per lookup against how full the table is
```

### Running your own benchmarks

`ob_bench` runs one book on one table per launch. To compare books, run it once per book:

```sh
# one book
./build/ob_bench --feed results/feed_itch_AAPL.bin --book map --table mix

# two books, same table
for book in map reverse_branchless; do
    ./build/ob_bench --feed results/feed_itch_AAPL.bin --book "$book" --table mix
done

# one book, five separate launches, flags the result if the medians differ by more than 5%
python3 scripts/repeat_runs.py --runs 5 ./build/ob_bench --feed results/feed_itch_AAPL.bin --book map --table mix
```

`ob_bench` options:

| option | default | what it does |
| --- | --- | --- |
| `--feed PATH` | required | the feed file to replay |
| `--book NAME` | `stub` | `stub`, `ref`, `table`, `map`, `vector`, `reverse_lb`, `reverse_branchless`, `reverse_linear` |
| `--table NAME` | `std` | `std`, `identity`, `mix` |
| `--limit N` | whole feed | replay only the first N updates |
| `--repeat N` | 1 | replay the whole feed N times, each time on a new empty book, and print each pass's median |
| `--block auto\|N` | `auto` | how many updates are timed together. `auto` uses 1 if the CPU's counter is precise enough, otherwise enough updates that one counter tick is a small part of the time |
| `--reserve N` | feed's peak live orders | how many orders the table is sized for |
| `--id-stride N` | 1 | multiply every order ID by N before replaying |
| `--prefetch` | off | start loading the next update's table slot into the cache while the current update runs |
| `--prefault N` | 0 | before timing, make and free N small allocations so that memory is already set up |
| `--scramble N` | 0 | before timing, make N small allocations of random sizes and free them in random order, so the book's allocations land in scattered places |
| `--no-aslr` | off | restart the program with memory at the same addresses every run |
| `--name LABEL` | built from book and table | the label used in the output files |
| `--summary PATH` | `results/summary.csv` | add one row of results to this file. `""` skips it |
| `--hist PATH` | `results/hist_<name>.csv` | write the timing histogram here |
| `--bucket-ns F` | 1 | width of each histogram bar, in ns |
| `--raw PATH` | off | write every timing sample to this file |
| `--no-dither` | off | don't spread out samples that fall exactly on counter ticks in the histogram |
| `--allow-noninvariant` | off | run even if the CPU's counter doesn't tick at a fixed rate |

Other tools:

- `ob_itch --in FILE.gz --symbols AAPL,MSFT [--out-dir DIR] [--from HH:MM] [--to HH:MM]`: writes one feed per symbol, optionally only for that time window
- `ob_genfeed --out PATH [--seed N] [--count N] [--new-level-rate F] [--levels N] [--add PCT --modify PCT --delete PCT] [--max-volume N]`: writes a synthetic feed. Defaults: seed 1, 10M updates, 1000 price levels per side, 5% of adds at a new price, 40% add, 30% modify, 30% delete
- `ob_validate --feed PATH`: checks that every modify and delete refers to a live order and the bid and ask never cross
- `ob_diff --feed PATH [--book NAME|all] [--table NAME|all] [--every N] [--depth N]`: compares a book against the reference book. Checks the top N price levels (default 8) after every update and the whole book every N updates (default 1000)


<br>
<h4>If you're confused by any term within the README, its definition can (most likely) be found in "Terms" below!</h4>
<details>
<summary><h2>Terms</h2></summary>

- **Order book**: the list of all buy orders (bids) and sell orders (asks) waiting to be filled for one stock
- **Price level**: all orders at one price on one side, kept as a single total volume
- **Best price / top of book**: the highest buy price and the lowest sell price
- **Order ID**: the number the exchange gives each order. Modify and delete messages carry only this number, not the price
- **ITCH**: Nasdaq's message format for every order added, changed, or removed on the exchange
- **Cycle counter**: a counter built into the CPU that ticks at a fixed rate. Reading it before and after a group of updates gives how long they took
- **Median/tail**: the median is the middle timing. The tail is the slowest timings, such as the slowest 1%
- **Heap allocation**: the program asking for more memory while it runs, for example with `new` or when a container grows. It is slow and its time varies, so we count it
- **cache line**: a small, fast memory next to the CPU. Data is copied into it in 64-byte pieces called cache lines
- **cache miss**: the data the CPU needs isn't in the cache, so it fetches from the main memory
- **Page fault**: the first write to a new 16 KB chunk of memory, where the operating system has to set that memory up before the write can finish
- **Hash table**: stores items in an array, using a formula on the order ID (the hash) to pick which position each one goes in
- **Slot**: one position in the hash table's array
- **Collision**: two order IDs that the formula sends to the same slot
- **Flat table**: an array of {order ID, order data} pairs, where each order sits at the slot its ID hashes to, or the first free slot after it.
- **Open addressing**: when two IDs want the same slot, the second one goes in a different slot of the same array
- **Linear probing**: when a slot is taken, try the next slot, then the one after, until an empty one is found
- **Modulo as a mask**: when the table size is a power of two, the remainder `id % size` equals `id & (size - 1)`, which the CPU does in one step instead of a slow division
- **Load**: filled slots divided by total slots. 0.5 means half full
- **Backward-shift deletion**: after removing an order, move the orders after it back one slot at a time to close the gap, so searches still find them and no "deleted" markers are left behind
- **Prefetch**: telling the CPU to start copying data into the cache before the code asks for it
- **`lower_bound`**: the standard library function that searches a sorted list by checking the middle and throwing away the half that can't contain the value, repeating until one element is left
- **Branchless search**: the same halving search written without `if` statements, so the CPU never has to guess which way an `if` goes
- **Branch predictor**: the part of the CPU that guesses which way each `if` goes so it can keep working ahead. A wrong guess wastes about 10 to 20 clock ticks
- **Warmup**: the first ~1% of samples, left out of the results because at the start the cache is empty, new memory is still being set up, the CPU hasn't learned which way the `if`s usually go, and the clock speed may still be rising
- **ASLR**: the operating system putting the program's memory at random addresses each run. Turned off here so memory is laid out the same every run
</details>

## TODO

- Re-run the milestone 3 numbers on AC power with nothing else running
- Try a third-party flat hash map (absl, ankerl or boost) next to the custom one, as a second point of reference
- Measure where updates actually land in the book and how far the linear search really walks, then tune the synthetic feed generator to match the real distributions
- The small optimizations from the talk, each with a before and after histogram
- Re-measure on x86 Linux with GCC, where the branchless search is expected to beat `lower_bound`
- Get every configuration inside the 5% spread gate


