# veloce - the unfriendly but fast node.js worker

| Task name | Throughput average (ops/s) | Throughput median (ops/s) | Latency average (ns) | Latency median (ns) | Samples |
| --------- | -------------------------- | ------------------------- | -------------------- | ------------------- | ------- |
| pool      | 1949411 ± 0.05%            | 2178649                   | 874.99 ± 10.37%      | 459.00              | 1147565 |
| piscina   | 65491 ± 0.20%              | 69367                     | 21914.67 ± 8.75%     | 14416.00            | 45632   |

```js
import { Worker } from "veloce";

const veloce = new Worker(new URL("./add.worker.js", import.meta.url));

const str1 = "Hello";
const str2 = "World";

veloce.run(
  // Reserve sufficient dst space. Assume 1 byte strings + 1 byte size prefix.
  str1.length + 1 + str2.length + 1,
  // Write synchronously!
  // Don't write out of bounds: dst.byteOffset + dst.byteLength!
  (dst) => {
    // Buffer serialize request.
    dst.buffer[dst.byteOffset] = dst.buffer.write(str1, dst.byteOffset + 1, str1.length, 'ascii');
    dst.byteOffset += dst.buffer[dst.byteOffset] + 1;
    dst.buffer[dst.byteOffset] = dst.buffer.write(str2, dst.byteOffset + 1, str2.length, 'ascii');
    dst.byteOffset += dst.buffer[dst.byteOffset] + 1;
  },
  // Read synchronously!
  (err, src) => {
    if (err) {
      console.error(err);
    } else {
      // Buffer deserialize response. 
      const str3 = src.buffer.toString(src.byteOffset + 1, src.buffer[src.byteOffset])); // "Hello World"
      console.log(str3);
    }

    veloce.destroy();
  },
);
```

```js
export default function (src, respond) {
  // Buffer deserialize request. 
  const str1 = src.buffer.toString(src.byteOffset + 1, src.buffer[src.byteOffset]); // "Hello"
  src.byteOffset += src.buffer[src.byteOffset] + 1;
  const str2 = src.buffer.toString(src.byteOffset + 1, src.buffer[src.byteOffset]); // "World"
  src.byteOffset += src.buffer[src.byteOffset] + 1;
  
  // Respond synchronously!
  respond(
    // Reserve sufficient dst space. Assume 1 byte strings.
    str1.length + str2.length + 1,
    // Write synchronously!
    // Don't write out of bounds: dst.byteOffset + dst.byteLength!
    // No exceptions!
    (dst, str1, str2) => {
      const str3 = str1 + ' ' + str2;
      // Buffer serialize response.
      dst.buffer[dst.byteOffset] = buffer.write(str3, dst.byteOffset + 1, str3.length, 'ascii'); // "Hello World" 
      dst.byteOffset += dst.buffer[dst.byteOffset] + 1;
    },
    // Only 3 args supported!
    str1,
    str2
  );
}
```
