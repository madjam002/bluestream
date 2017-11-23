# bluestream

[![Build Status](https://travis-ci.org/bustle/bluestream.svg?branch=master)](https://travis-ci.org/bustle/bluestream) [![Try bluestream on RunKit](https://badge.runkitcdn.com/bluestream.svg)](https://npm.runkit.com/bluestream)

A collection of NodeJS Streams and stream utilities that work well with promises and async functions. Think `through2-concurrent` with promise support. The goal is to reduce the edge cases when mixing streams and promises. These are a little slower than normal streams but much more forgiving.

Originally forked from [promise-streams](https://github.com/spion/promise-streams) but with some different goals and a lot more tests. Named after bluebird but not actually using bluebird. (Though we work great with it, highly recommended!)

- `ReadStream` Easy async producing of data
- `TransformStream` Easy async transforming of data
- `WriteStream` Easy async writing of data
- `FilterStream` similar to `Array.prototype.filter` Easy stream filtering of data
- `ReduceStream` similar to `Array.prototype.reduce` but a stream that emits each step and `.promise()` resolves to the end result

- `bstream.wait(stream)` resolves when the stream finishes
- `bstream.collect(stream)` Concats strings and buffers, returns an array of objects.
- `bstream.pipe(source, target, [target,])` Returns a promise for when the last target stream finishes

# Examples

```js
const bluestream = require('bluestream')
const got = require('got')

const pokeStream = bluestream.read(async function () {
  this.offset = this.offset || 0
  const { body: pokemon } = await got(`https://pokeapi.co/api/v2/pokemon/?offset=${this.offset}`, { json: true })
  if (pokemon.results.length > 0) {
    this.offset += pokemon.results.length
    pokemon.results.map(monster => this.push(monster))
  } else {
    return null
  }
})

const pokedexStream = bluestream.transform({ concurrent: 2 }, ({ url }) => got(url, { json: true }).then(resp => resp.body))
const logStream = bluestream.write(pokemon => console.log(pokemon.name, pokemon.sprites.front_default))

await bluestream.pipe(pokeStream, pokedexStream, logStream)
console.log('caught them all')
```

# api

#### ps.read

`([opts:Options,] fn:(bytesWanted) => Promise)) => ReadStream`

#### ReadStream

Create a read-promise stream. Pass it a function that takes the number of bytes or objects of wanted data and and uses `this.push` or `return` to push values or promises. This function should return a promise that indicates when the object/chunk are fully processed. Return `null` to end the stream.

Options:
  * `read` - An optional way to pass the read function

  * `objectMode` - true or false

  * all other `Readable` stream options

The other options are also passed to node's Read stream constructor.

A `ReadStream` works like a normal `ReadableStream` but the `_read` and `push()` methods have some notable differences. (The `_read` method can be provided as the only argument, in a `read` key on the options, or as the `_read` method if you extend `ReadStream`.) Any returned, non undefined, value will automatically be pushed. Object mode is the default.

`_read(bytesWanted)`
- Is async function friendly, a rejection/throw will be handled as an error event
- Is called again only after it returns or resolves regardless of how many times you call `.push`
- Is called again if you don't push (To aid in control flow)
- Pushes any non `undefined` return values

`this.push()`
- Can be pushed a promise
- Returns a promise

This allows you to use it in some friendly ways.

```js
// readable stream from an array
const list = [1, 2, 3]
const listStream = bstream.read(() => list.shift() || null)

// readable stream from redis scans
import Redis from 'io-redis'
const redis = new Redis()
let cursor = 0

const hscanStream = bstream.read(async () => {
  const [newCursor, keys] = await redis.scan('cursor', cursor)
  keys.map(key => this.push(key))
  if (newCursor === '0') {
    this.push(null)
  }
  cursor = newCursor
})
```

#### ps.transform
#### ps.map

`([opts:Options,] fn:(data[, enc]) => Promise)) => TransformStream`

#### TransformStream

Create a transform-promise stream. Pass it a function that takes data and
encoding and uses `this.push` to push values or promises. Any returned, non undefined, value will automatically be pushed. This function should
return a promise that indicates when the object/chunk are fully processed.

Options:
  * `transform` - An optional way to pass the transform function

  * `concurrent` - The maximum number of concurrent promises that are allowed.
    When this limit is reached, the stream will stop processing data and will
    start buffering incoming objects. Defaults to `1`

  * `highWatermark` - the size (in objects) of the buffer mentioned above. When
    this buffer fills up, the backpressure mechanism will activate. Its passed
    to node's transform stream.

The other options are also passed to node's Transform stream constructor.

#### ps.write

`([opts:Options,] fn:(data[, enc]) => Promise)) => WriteStream`

#### WriteStream

Create a write-promise stream. Pass it a function that takes data and
encoding returns a promise that indicates when the object/chunk are fully processed.

Options:
  * `write` - An optional way to pass the write function

  * `writev` - Not supported, and passed directly to the underlying `Writable` stream

  * `concurrent` - The maximum number of concurrent promises that are allowed.
    When this limit is reached, the stream will stop processing data and will
    start buffering incoming objects. Defaults to `1`

  * `highWatermark` - the size (in objects) of the buffer mentioned above. When
    this buffer fills up, the backpressure mechanism will activate. Its passed
    to node's write stream.

The other options are also passed to node's Write stream constructor.

#### ps.filter

`([opts:Options,] fn: async (data[, enc]) => boolean) => FilterStream`

Create a new FilterStream. The function should return a boolean to
indicate whether the data value should pass to the next stream

Options: Same as `ps.transform`

#### ps.reduce

`([opts:Options,] fn: (acc, data[, enc]) => Promise) => ReduceStream`

Reduces the objects in this promise stream. The function takes the resolved
current accumulator and data object and should return the next accumulator
or a promise for the next accumulator.

The ReduceStream has a `promise()` method which returns the final
accumulator value

```js
process.stdin.pipe(split()).pipe(es.reduce(function(acc, el) {
    return acc + el;
})).promise().then(function(sum) {

});
```

#### ps.wait

`(s: Stream) => Promise`

Wait for the stream to end. Also captures errors.

#### ps.pipe

`(source: Stream, destination: Stream) => Promise`

Pipes s1 to s2 and forwards all errors to the resulting promise. The promise is
fulfilled without a value when the destination stream ends.

#### ps.collect

`(source: Stream) => Promise`

Returns a Buffer, string or array of all the data events concatenated together. If no events null is returned.

#### PromiseStream.promise

`() => Promise`

Returns a promise fulfilled at the end of the stream, rejected if any errors
events are emitted by the stream.

For `ReduceStreams`, the promise is for the final reduction result. Any
stream errors or exceptions encountered while reducing will result with a
rejection of the promise.
