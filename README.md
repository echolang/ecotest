# ecotest

Checks and test events for `echoc test`. Failures name the test's line, and the same repo ships the runner that folds those events, coverage, skips and flakes into one report.

- `expect($x)->is($want)` for strings, numbers, lists, maps, results, nullables, and your own types.
- A failed check prints expected vs got. Multi-line text gets a line diff.
- Soft checks hold every failure until the scope ends, then report them together.
- `context("row {$i}")` rides on every failure while the value lives.
- Snapshots live next to the test file. `ECOTEST_CI=1` refuses to write new ones.
- A property test shrinks a failing value and prints `ECOTEST_SEED` so you can replay it.
- `ecotest` wraps `echoc test` and reports coverage, gates, history and flakes.
- HTTP checks against a URL, a service the tool started, or a nexus app in process. Same tests.

`echoc test` still runs the checks. The events are for the tool. Without a run directory, nothing is written and every check still works.

## Installation

From your project directory:

```bash
epm add echolang/ecotest --git https://github.com/echolang/ecotest --range ^0.3
```

The require belongs on the test target. The library is standard library only, so any module can add it there. The CLI, JSON and HTTP libraries compile into the `ecotest` binary. Your tests stay on the standard library.

```echo
#[target: test] {
    #[sources: "tests/**/*.eco"]
    #[requires: "echolang/ecotest" { version: "^0.3", source: git "https://github.com/echolang/ecotest" }]
}
```

Then `epm run ecotest`. From this repo, `echoc install` puts `ecotest` on `PATH`.

## Quick start

```echo
use ecotest::expect;

test the_list_holds_both
{
    array<string> $names = ['ada', 'bob'];
    expect($names)->is(['ada', 'bob'])->has('bob')->lacks('cy');
}
```

```bash
echoc test
```

A failed check stops at the test's line and prints what was expected and what came instead. That is the whole loop until you want coverage, skips, snapshots in CI, or a report: then `ecotest`.

## Checks

`expect` picks an overload from the value. Chain the checks. Every one names the test's line.

### Numbers

Signed widths widen to `int64`, unsigned (including `usize`) to `uint64`. `is` on a float is exact. Use `near` for anything computed.

```echo
expect(42)->is(42)->above(41)->below(43)->between(40, 50);
expect(0.5)->is(0.5);
expect(0.1 + 0.2)->near(0.3);
```

### Bools

Bools are `yes` and `no`. `says:` names the condition, because the expression that produced the bool is gone by the time the check runs.

```echo
expect(true)->yes();
expect($bound)->yes(says: 'the listener bound');
```

### Text

```echo
expect('hello Ada')->is('hello Ada')->has('Ada')->lacks('Bob');
expect('hello Ada')->starts('hello')->ends('Ada')->length(9);
expect('')->empty();
```

When either side spans several lines, `is` prints a line diff instead of quoting both strings.

### Lists and maps

`is` on a list is the same items in the same order. `same` is the same items, any order. A failed map `is` lists missing, extra and changed keys.

```echo
array<string> $names = ['ada', 'bob'];
expect($names)->is(['ada', 'bob'])->size(2)->has('bob')->lacks('cy');
expect($names)->same(['bob', 'ada'])->sorted();
expect($names)->all((const string& $n) => !$n->empty(), says: 'named');

map<string, int32> $ages = ['ada' => 36, 'bob' => 41];
expect($ages)->has(key: 'ada')->at('bob', 41)->size(2);
expect($ages)->is(['bob' => 41, 'ada' => 36]);
```

`sorted` needs `<` on the item type. Checks that compare items need `==`, and they print them with `str::from`.

### Results

`ok()` hands back the value. If it was an error, the test stops, even inside `soft()`, because there is nothing to go on with. `fails()` is the other direction. `status` has `fine()` for success. A nullable has `some()` and `none()`.

```echo
int32 $n = expect(parse('7'))->ok();
expect($n)->is(7);
expect(parse('8'))->ok(8);
expect(parse(''))->fails(Fault::empty);

expect(check(true))->fine();
expect(find(true))->some(3);
expect(find(false))->none();
```

`E` needs `str::from` so an error can print. For an error enum whose cases carry values, take `fails()` and check what it returns.

### Your own types

The catch-all is labelled, so `expect($list)` still means the list checks:

```echo
expect(value: Point(1, 2))->is(Point(1, 2))->isNot(Point(2, 1));
expect(value: Point(1, 2))->satisfies((const Point& $p) => $p->x < $p->y, says: 'rising');
```

Comparing needs `==` on `T`. Printing a failure needs `str::from`.

## Soft checks

Bind the value. If you don't, the destructor runs on that statement and the scope is already over.

```echo
$all = ecotest::soft();
expect($got->name)->is('Ada');
expect($got->id)->is(1);
```

Failed checks are held until `$all` goes, then reported together. The test stops at the first one's line.

`context` is the same shape. Bind it inside a loop so every failure names the row:

```echo
for (usize $i = 0; $i < $rows->count(); $i++) {
    $at = ecotest::context("row {$i}");
    expect($rows[$i]->id)->above(0);
}
```

### Skip

`skip` exits 0, so a plain `echoc test` counts the test as passed. The tool reads the `skip` event and reports it as skipped.

```echo
ecotest::skip('windows only');
string $url = ecotest::needs(env: 'MYSQL_TEST_URL');
```

`needs` is skip-if-unset, for tests that talk to a live service.

## Snapshots

```echo
expect($rendered)->snapshot();
```

The n-th snapshot a test takes lives at `<test file dir>/__snapshots__/<file stem>/<test name>.<n>.snap`, so it moves with the file and test that own it.

A missing file is written and reported. A changed file fails with a diff. `ECOTEST_UPDATE=1` rewrites the ones that changed. `ECOTEST_CI=1` refuses to write a new one.

Snapshots need `ECHO_TEST_FILE` and `ECHO_TEST_NAME`, which `echoc test` sets. A snapshot taken outside a test run fails with that reason.

## Properties

`forAll` draws values from a generator, checks `$holds`, and on failure shrinks to the smallest value that still fails. Default 200 runs.

```echo
use ecotest::gen;

ecotest::forAll(gen::ints(-100, 100), (const int64& $n) => $n + 0 == $n);
ecotest::forAll(gen::lists(gen::ints(0, 9)), (const array<int64>& $xs) => $xs->count() <= 16);
ecotest::forAll(gen::text(max: 8), (const string& $s) => $s->size() <= 8);
```

The failure names the smallest value, the first failing value, and `ECOTEST_SEED` so the next run replays it. The tool sets the seed for the run; you can set it yourself.

`gen` also has `floats`, `bools`, `pick`, `pairs`, and `map` (push values through a function; a mapped value cannot be shrunk back).

## Fixtures

`$dir = TempDir();` at the top of a test is the whole setup. The directory is gone when the value goes. `read` answers a `result` with the `std::io::ioerror`, so a missing file is `fails()` and its contents are `ok(...)`; a directory or file that cannot be made stops the test at your line.

```echo
use ecotest::TempDir;

$dir = TempDir();
$dir->write('a.txt', 'hello');
expect($dir->read('a.txt'))->ok('hello');
expect($dir->read('missing.txt'))->fails();
```

`Calls<T>` is a recorder a fake and the test share. Record inside the fake, check afterwards:

```echo
use ecotest::Calls;

$calls = Calls<string>();
$notify = function(string $who) : void {
    $calls->record($who);
};

$notify('ada');
$notify('bob');
expect($calls)->times(2)->nth(0, 'ada')->with('bob');
```

`eventually` polls until the closure is true, then fails. For a server binding its port, a worker draining a queue:

```echo
ecotest::eventually(() : bool => $server->ready());
```

`ecotest::run` starts a program and captures both streams. `expect($ran)->succeeds()->stdout()->has('done')` reads as one sentence. A program that cannot start stops the test. `$ran->code` is the exit code, or null when a signal ended the program.

```echo
$ran = ecotest::run('echo', ['hello']);
expect($ran)->succeeds()->stdout()->is("hello\n");
expect($ran)->stderr()->empty();
expect($ran->code)->some(0);
```

`bench` times a closure, finds a batch that takes about a millisecond, then samples until the budget is spent. Median and p95 go on a `bench` event. `note` and `metric` are the same channel: the tool shows them under the test.

```echo
ecotest::note('ran against the fixture set');
ecotest::metric('rows', 12);
ecotest::bench('add', function() : void {
    int64 $sum = 0;
    for (int64 $i = 0; $i < 100; $i++) {
        $sum = $sum + $i;
    }
});
```

## The runner

`ecotest` runs `echoc test`, reads the events the tests wrote, and turns that into one run. Runs live under `.ecotest/run-<millis>/`, with `history.jsonl` beside them, outside `ecobuild/` so `echoc clean` keeps the history.

```bash
epm run ecotest --coverage
ecotest --coverage --html --min-coverage 80
```

### Commands

- `run`, the default when no command is named, compiles, runs, prints a summary, writes the run directory, applies the gates.
- `report [run] --format terminal|md|junit|lcov|html|json` renders a stored run again.
- `history` is duration and outcome trends.
- `api` lists public functions and methods the newest coverage run never called.
- `who <file:line>` names the tests that ran that source line.
- `affected --since <ref>` runs only the tests whose coverage touches lines changed since a git ref.

`--repeat N` runs every test N times and flags the ones whose passes disagree. `--filter` and `--define` and `--target` and `--timeout` go through to `echoc test`. `--style auto` draws live progress on a terminal and a styled summary when stdout takes colour; `pretty` and `simple` pin it.

`--min-coverage`, `--min-branch-coverage`, `--min-diff-coverage` and `--max-flaky` fail the run when a threshold does not hold. `--since` is the git ref for diff coverage. `--html` writes `report.html` into the run directory. `ecotest --help` is the rest of the flags.

### `ecotest.json`

Optional, beside the manifest. Flags on the command line win.

```json
{
  "coverage": {
    "exclude": ["src/gen/**"],
    "modules": ["app"],
    "min": 80
  },
  "services": [
    {
      "name": "api",
      "run": ["./build/server", "--port", "8080"],
      "ready": { "http": "http://127.0.0.1:8080/health", "timeout_ms": 10000 }
    }
  ]
}
```

The tool starts each service, waits until it is ready, exports `ECOTEST_<NAME>_URL` (upper case, non-alphanumerics as `_`), and stops them after the tests. Output goes to `<run>/services/<name>.log`. A failing run prints the end of each log. `ready` can also be `{ "ms": 200 }` when there is nothing to probe.

## HTTP

`modules/http` is a second package in this git repo. Same tests talk to a deployed URL, a service the tool started, or an app dispatched in process.

```echo
#[requires: "echolang/ecotest/modules/http" { version: "^0.3", source: git "https://github.com/echolang/ecotest" }]
```

One `Api`, plus `live` for a service the tool started:

```echo
use ecotest::http::{Api, live};

$api = Api('http://127.0.0.1:8080');
$api = live(service: 'api');
$api = Api(transport: $transport);
```

`live(service:)` reads `ECOTEST_API_URL`. Outside `ecotest` that variable is unset and the test skips.

```echo
$api->get('/users/1')
    ->status(200)
    ->ok()
    ->type('application/json')
    ->json('$.name', 'Ada')
    ->json('$.id', 1)
    ->present('$.items[0].id')
    ->count('$.items', 2)
    ->faster(than: 200);

$id = $api->post('/users', json: '{"id":7,"name":"Bob"}')
    ->status(201)
    ->capture('$.id');

$api->bearer('t0k')->get('/whoami')->body('Bearer t0k');
$api->get('/users/1')->snapshot(['$.items[*].at']);
```

`post`, `put` and `patch` take the body by its encoding: `json:` text or a `json::Value`, `form:` fields (a `map<string, string>`, URL-encoded), or `body:` with `type:`. Each sends its own Content-Type.

JSON paths are `$.items[0].name`, `$.items[*].id`, `$['odd key']`. The leading `$` is optional. `[*]` fans out over an array, so a path can name several values.

A failed check shows the exchange: the request line, the status, the headers, the start of the body. A request that got no response at all fails the first check made on it, at the test's line.

A JSON snapshot is stored pretty-printed. Paths in `$redact` are replaced first, so ids and timestamps do not break it.

`Service` starts a program for one test and stops it when the value goes. Shared services belong in `ecotest.json` with `live(service:)`.

## This repo

```bash
echoc build
echoc test
echoc run --target ecotest -- --coverage
```

MIT. Copyright 2026 Mario Döring.
