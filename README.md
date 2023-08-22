# vcr-test

Record your test suite's HTTP interactions and replay them during future test runs for fast, deterministic, accurate tests.

## Installation

```bash
npm install vcr-test --save-dev
```

## Usage
The first time the test runs, it should make live HTTP calls. vcr-test will take care of recording the HTTP traffic and storing it. Future test runs replay the recorded traffic.

```js
import { join } from 'node:path';
import { VCR, FileStorage } from 'vcr-test';
import { api } from './my-api'


describe('some suite', () => {
  it('some test', async () => {
    // Configure VCR
    const vcr = new VCR(new FileStorage(join(__dirname, '__cassettes__')));

    // Intercept HTTP traffic
    await vcr.useCassette('cassette_name', async () => {
      const result = await api.myAwesomeApiCall();
      expect(result).toBeDefined();
    });
  })
})
```

## Extensibility

### Request Matching
When running a test the library will try to find a match in the specified cassette that matches on url, headers, and body. However, you may want to change this behavior to ignore certain headers and perform custom body checks.

The default request matcher allows you to change some of its behavior:

```ts
const vcr = new VCR(...);

// the request headers will not be compared against recorded HTTP traffic.
vcr.matcher.compareHeaders = false; 

// the request body will not be compared against recorded HTTP traffic.
vcr.matcher.compareBody = false;

// This will ignore specific headers when doing request matching
vcr.matcher.ignoreHeaders.add('timestamp');
```

Alternatively, you can extend the default request matcher:

```ts
import { DefaultRequestMatcher } from 'vcr-test';

class MyCustomRequestMatcher extends DefaultRequestMatcher {
  public bodiesEqual(recorded: HttpRequest, request: HttpRequest): boolean {
    // custom body matching logic
  }

  public headersEqual(recorded: HttpRequest, request: HttpRequest): boolean {
    // custom headers matching logic
  }

  public urlEqual(recorded: HttpRequest, request: HttpRequest): boolean {
    // custom url matching logic
  }

  public methodEqual(recorded: HttpRequest, request: HttpRequest): boolean {
    // custom method matching logic
  }
}
```

If you have more advanced matching needs you can implement your own Request Matcher:

```ts
/**
 * Matches an app request against a list of HTTP interactions previously recorded
 */
export interface IRequestMatcher {
  /**
   * Finds the index of the recorded HTTP interaction that matches a given request
   * @param {HttpInteraction[]} calls recorded HTTP interactions
   * @param {HttpRequest} request app request
   * @returns {number} the index of the match or -1 if not found
   */
  indexOf(calls: HttpInteraction[], request: HttpRequest): number;
}
```

and assign the custom implementation like this:

```ts
const vcr = new VCR(...);
vcr.matcher = new MyCustomRequestMatcher();
```

For more details refer to the [DefaultRequestMatcher](https://github.com/epignosisx/vcr-test/blob/main/src/default-request-matcher.ts) implementation.

### Storage
The library comes with a File storage implementation that saves files in YAML for readibility. However, you may prefer to save the cassettes in a database and in JSON. You can change the storage and file format by creating a different storage implementation.

This is the interface you need to satisfy:

```ts
/**
 * Cassette storage
 */
export interface ICassetteStorage {
  /**
   * Loads a cassette from storage or undefined if not found.
   * @param {string} name cassette name
   * @returns {Promise<HttpInteraction[] | undefined>}
   */
  load(name: string): Promise<HttpInteraction[] | undefined>;

  /**
   * Saves HTTP traffic to a cassette with the specified name
   * @param {string} name cassette name
   * @param {HttpInteraction[]} interactions HTTP traffic
   * @returns {Promise<void>}
   */
  save(name: string, interactions: HttpInteraction[]): Promise<void>;
}
```

Then just initialize VCR with your implementation:

```ts
const vcr = new VCR(new DatabaseStorage());
```

For more details refer to the [FileStorage](https://github.com/epignosisx/vcr-test/blob/main/src/file-storage.ts) implementation.

