# Motivation
Tarantool (memtex) is an in-memory engine and It doesn't waste time to
locks and synchronizations. Moreover, Tarantool can avoid writes to a
disk.

Tarantools advantages:
- no locks
- no random disk writes

Tarantools disadvantages:
- single threaded
- any iteration over index is random reads from RAM (tuples are
  allocated in slabs)

There is an assumption that this architecture should help beat ordinary
SQL databases.

# Purposes of research
- find performance bottlenecks
- compare the speed of Tarantool SQL with other DBs on different
  workloads
- improve Tarantool andbeat other DBs

# Repo structure
In this repository, there is a tree-structured set of folders which
repeat the structure of `Test Plan` section. In each of those folders
one can find instructions to repeat 

# Test Plan
- [ ] enable LTO (plan described below) _(it can disclose a real
  bottle-neck)_
- [ ] btree bench (plan described below)
- [ ] sql - engine bench
    - [ ] compare speed to `C` implementation of the same query
      - [x] write tests
          - [x] huge select (~20% slowdoun)
          - [ ] small select (~50% slowdoun)
      - [ ] process results and put it here
    - [ ] investigate [existing benchmarks](https://github.com/tarantool/tarantool/wiki/SQL-performance-benchmarks)
        - [ ] check if bench is written correc for tarantool
- [ ] io bench
    - [ ] select over `net.box`
    - [ ] research fsync influence
        - [ ] workload with lots of fibers
        - [ ] workload with a single fiber
    - [ ] nginx butching bench (possibly main perf improvement)
- [ ] real-world bench
    - [ ] steal a workload from production and analyze patterns and
      bottlenecks

# Btree bench plan
- [ ] randomization influence
  - [x] absolute random (main consumer - cache misses)
  - [ ] sequential (main consmer - msgpuck/btree call stack?)
    - [ ] compare to hash index
  - [ ] mixed
    - [ ] find `main consumer changed` points
- [ ] workload influence
  - [ ] lots of small requests (part of `randomization influence`)
  - [ ] `select join`
    - [ ] huge join
    - [ ] join + index
      - [ ] w/o index
      - [ ] with index
- [ ] hint patch investigation (store data for comparition straight in
  btree)
  - [ ] btree research
    - [ ] block size, block traversal (current 512b + lineral traversal
      for hints)
        - [x] binary search is faster? `No`
        - [x] other block size for lineral travversal is faster? `No`
- [ ] prefetch operation influence (async work with memory)
    - [ ] write asyny-memory `hello_rowld.c` test

# Enable LTO plan
Tarantool fails to start when compiled with LTO because linker ignores
`-Wl,--dynamic-list,${exports_file}` option
- [x] create small prog to repeat exports problem
  - [x] see that LTO works
  - [x] see that on LTO enable exports disappear
  - [x] try to build with the gold linker (the same result)
  - [x] build with `-rdynamic` (exports preserved, binary changed)
- [x] compile with clang (the same result)
- [x] build with lto and `-rdynamic` (speed decreased?)
  - [X] write a [cpu/memory bound test which measures performance on different revisions](https://gist.github.com/Khatskevich/31a2da6ab46ce903120e7a03d65966db)
- [x] check `__attribute__((used))` (dynamic-list start working!) -> problem possibly in ld
- [x] Write [minimal problem reproducer](https://gist.github.com/Khatskevich/54771081aefbc8420d79fdcfb65d2662), which shows the export-lto-used problem
- [ ] Fix cmake to build with lto in case of new binutils
- [ ] Fix lto for mac
- [ ] Find out why gcc slows down?
    - [ ] `O2` -> `O3`


# Known problems
- sql
  - lots of `memcpy` and `malloc` in SQL engine
    - ephemeral tables work inefficiently
    - extra memcpy's on inserts to any table
  - the absence of prepared statements (query recompiling on each
    request)
  - the same core for query compilation and execution
- architecture
  - single threaded engine
  - single-threaded scan
  - sequential scan -> random reads (cache locality is not used)
    - _minor_ not covering b-tree indexes (random reads on any
      comparison)
- slow tuple compare
  - huge stack of calls
  - slow msgpuck


# Related links
- [lua tables faster then Tarantool spaces 250x (interesting sequential scan test results)](https://github.com/tarantool/tarantool/issues/458#issuecomment-53895929)
