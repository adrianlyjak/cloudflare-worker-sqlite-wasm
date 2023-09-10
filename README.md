A fork of [sql.js](https://github.com/sql-js/sql.js) with some minor modifications to support running in memory
sqlite databases in cloudflare workers.

1. The standard builds of sql.js that are generated try to infer "web worker" vs "web" environment via emscripten. 
   Unfortunately, cloudflare workers are detected as a web worker, and then start throwing errors from other 
   global fields missing when trying to initialize within cloudflare.
2. Standard emscripten tries to bootstrap the worker from a dynamically loaded file. Cloudflare workers don't 
   support initializing web workers from binary, and rather need to directly `import` the web assembly module.
   This can be supported with an `instantiateWasm` config on the emscripten generated layer so long as the




```js
// functions/helloworld.js
import wasm from "../sqljs/sql-wasm.wasm";
import initSqljs from "../sqljs/sql-wasm";

export async function onRequest(context) {
  try {
    const SQL = await initSqljs({
      instantiateWasm(info, receive) {
        let instance = new WebAssembly.Instance(wasm, info);
        receive(instance);
        return instance.exports;
      },
      log: console.log,
      error: console.error,
    });

    const db = new SQL.Database();
    let sqlstr =
      "CREATE TABLE hello (a int, b char); \
INSERT INTO hello VALUES (0, 'hello'); \
INSERT INTO hello VALUES (1, 'world');";
    db.run(sqlstr); // Run the query without returning anything

    // Prepare an sql statement
    const stmt = db.prepare("SELECT * FROM hello WHERE a=:aval AND b=:bval");

    // Bind values to the parameters and fetch the results of the query
    const result = stmt.getAsObject({ ":aval": 1, ":bval": "world" });
    console.log(result); // Will print {a:1, b:'world'}
    return new Response(JSON.stringify(result), { status: 200 });
  } catch (ex) {
    console.error("failed to start", ex);
    return new Response("ouch, ", { status: 500 });
  }
}


```