<!--
author:  André Dietrich

email:   LiaScript@web.de

version: 0.0.4

logo:    logo.png

edit:    true

comment: PGlite Template for LiaScript.

@onload
window.databases = window.databases || {}

if(!window.PGlite) {
    const urls = [
        "https://cdn.jsdelivr.net/npm/@electric-sql/pglite/dist/index.js",
        "https://cdn.jsdelivr.net/npm/@electric-sql/pglite/dist/pgtap/index.js",
    ]

    Promise.all(urls.map(url => import(url)))
    .then((modules) => {
        const [PGlite, PGTap] = modules

        window.PGlite = PGlite.PGlite
        console.log("PGlite loaded", PGlite)

        window.PGTap = PGTap
        console.log("PGTap loaded", PGTap)
    })
    .catch((err) => {
        console.warn("PGlite not loaded", err.message)
    })
}

class AccessQueue {
  constructor() {
    this.queue = [];
    this.active = false;
  }

  async grantAccess() {
    if (!this.active) {
      this.active = true;
      return;
    }
    return new Promise(resolve => this.queue.push(resolve));
  }

  releaseAccess() {
    if (this.queue.length > 0) {
      const next = this.queue.shift();
      next();
    } else {
      this.active = false;
    }
  }
}

// Usage:
window.accessQueue = new AccessQueue();

window.getDatabase = function(id) {
    if (window.databases[id] === undefined) {
      window.databases[id] = new PGlite({
        extensions: { pgtap: window.PGTap.pgtap }
      })
    }
    return window.databases[id]
}

window.dbdiagramQuery = `WITH
tables AS (
  SELECT table_schema, table_name
  FROM information_schema.tables
  WHERE table_type = 'BASE TABLE'
    AND table_schema NOT IN ('pg_catalog', 'information_schema')
),

cols AS (
  SELECT
    c.table_schema,
    c.table_name,
    c.column_name,
    lower(
      CASE
        WHEN c.data_type IN ('character varying','character')
             AND c.character_maximum_length IS NOT NULL
          THEN 'varchar(' || c.character_maximum_length || ')'
        WHEN c.data_type IN ('numeric','decimal')
             AND c.numeric_precision IS NOT NULL
          THEN 'decimal(' || c.numeric_precision
               || COALESCE(',' || c.numeric_scale, '') || ')'
        WHEN c.data_type IN ('integer','int4') THEN 'int'
        WHEN c.data_type IN ('bigint','int8') THEN 'bigint'
        WHEN c.data_type IN ('smallint','int2') THEN 'smallint'
        WHEN c.data_type IN ('timestamp without time zone',
                             'timestamp with time zone',
                             'timestamp')
          THEN 'timestamp'
        WHEN c.data_type IN ('time without time zone',
                             'time with time zone',
                             'time')
          THEN 'time'
        ELSE c.data_type
      END
    ) AS data_type,
    c.is_nullable,
    c.ordinal_position
  FROM information_schema.columns c
  JOIN tables t USING (table_schema, table_name)
),

pk_cols AS (
  SELECT
    k.constraint_schema,
    k.table_schema,
    k.table_name,
    k.column_name
  FROM information_schema.table_constraints tc
  JOIN information_schema.key_column_usage k
    ON tc.constraint_name = k.constraint_name
   AND tc.constraint_schema = k.constraint_schema
  WHERE tc.constraint_type = 'PRIMARY KEY'
),

unique_cols AS (
  SELECT
    k.constraint_schema,
    k.table_schema,
    k.table_name,
    k.column_name,
    k.constraint_name,
    COUNT(*) OVER (PARTITION BY k.constraint_schema, k.constraint_name) AS cols_in_constraint,
    k.ordinal_position
  FROM information_schema.table_constraints tc
  JOIN information_schema.key_column_usage k
    ON tc.constraint_name = k.constraint_name
   AND tc.constraint_schema = k.constraint_schema
  WHERE tc.constraint_type = 'UNIQUE'
),

unique_multi AS (
  SELECT
    uc.constraint_schema,
    uc.table_schema,
    uc.table_name,
    uc.constraint_name,
    string_agg(quote_ident(uc.column_name), ', ' ORDER BY uc.ordinal_position) AS cols
  FROM unique_cols uc
  WHERE uc.cols_in_constraint > 1
  GROUP BY uc.constraint_schema, uc.table_schema, uc.table_name, uc.constraint_name
),

index_blocks AS (
  SELECT
    um.table_schema,
    um.table_name,
    '  indexes {' || E'\n' ||
    string_agg('    (' || um.cols || ') [unique]', E'\n') ||
    E'\n' || '  }' AS idx_block
  FROM unique_multi um
  GROUP BY um.table_schema, um.table_name
),

col_lines AS (
  SELECT
    c.table_schema,
    c.table_name,
    c.ordinal_position,
    '  ' || quote_ident(c.column_name) || ' ' || c.data_type ||
    CASE
      WHEN
        (
          (pk.column_name IS NOT NULL) OR
          (c.is_nullable = 'NO') OR
          EXISTS (
            SELECT 1 FROM unique_cols u
            WHERE u.table_schema = c.table_schema
              AND u.table_name   = c.table_name
              AND u.column_name  = c.column_name
              AND u.cols_in_constraint = 1
          )
        )
      THEN
        ' [' ||
        trim(both ' ' from
          concat_ws(', ',
            CASE WHEN pk.column_name IS NOT NULL THEN 'pk' END,
            CASE WHEN c.is_nullable = 'NO' THEN 'not null' END,
            CASE
              WHEN EXISTS (
                SELECT 1 FROM unique_cols u
                WHERE u.table_schema = c.table_schema
                  AND u.table_name   = c.table_name
                  AND u.column_name  = c.column_name
                  AND u.cols_in_constraint = 1
              ) THEN 'unique'
            END
          )
        ) || ']'
      ELSE '' END
    AS line
  FROM cols c
  LEFT JOIN pk_cols pk
    ON pk.table_schema = c.table_schema
   AND pk.table_name   = c.table_name
   AND pk.column_name  = c.column_name
),

table_blocks AS (
  SELECT
    'Table ' || quote_ident(t.table_name) || ' {' || E'\n' ||
    string_agg(cl.line, E'\n' ORDER BY cl.ordinal_position) ||
    COALESCE(E'\n' || ib.idx_block, '') || E'\n}'
    AS dbdiagram_block
  FROM tables t
  JOIN col_lines cl
    ON cl.table_schema = t.table_schema
   AND cl.table_name   = t.table_name
  LEFT JOIN index_blocks ib
    ON ib.table_schema = t.table_schema
   AND ib.table_name   = t.table_name
  GROUP BY t.table_schema, t.table_name, ib.idx_block
),

fk_map AS (
  SELECT
    k.constraint_schema,
    k.table_schema,
    k.table_name,
    k.column_name,
    ccu.table_schema  AS ref_schema,
    ccu.table_name    AS ref_table,
    ccu.column_name   AS ref_column
  FROM information_schema.table_constraints tc
  JOIN information_schema.key_column_usage k
    ON tc.constraint_name  = k.constraint_name
   AND tc.constraint_schema = k.constraint_schema
  JOIN information_schema.referential_constraints rc
    ON rc.constraint_name   = tc.constraint_name
   AND rc.constraint_schema  = tc.constraint_schema
  JOIN information_schema.constraint_column_usage ccu
    ON ccu.constraint_name   = rc.unique_constraint_name
   AND ccu.constraint_schema  = rc.unique_constraint_schema
  WHERE tc.constraint_type = 'FOREIGN KEY'
),

ref_lines AS (
  SELECT
    'Ref: ' || quote_ident(fk.table_name) || '.' || quote_ident(fk.column_name) ||
    ' > '  || quote_ident(fk.ref_table)  || '.' || quote_ident(fk.ref_column)
    AS ref_line
  FROM fk_map fk
)

SELECT
  (SELECT string_agg(dbdiagram_block, E'\n\n') FROM table_blocks)
  || CASE WHEN EXISTS (SELECT 1 FROM ref_lines) THEN E'\n\n' ELSE '' END
  || COALESCE((SELECT string_agg(ref_line, E'\n') FROM ref_lines), '')
  AS dbdiagram;
`

window.dbdiagram = function (dbml) {
    // Unicode-safe: UTF-8 → base64 → URL-encode
    const base64 = btoa(unescape(encodeURIComponent(dbml)));
    const c = encodeURIComponent(base64);

    return `<iframe
      src="https://dbdiagram.io/embed?c=${c}"
      width="100%"
      height="600px"
      style="border: 0"
      referrerpolicy="no-referrer"
      allowfullscreen
    ></iframe>
    <center><a style="font-size: smaller" target="_blank" href="https://dbdiagram.io/embed?c=${c}">dbdiagram.io</a></center>`;
}

window.renderReplHtml = function(responseOrResult, options) {
  options = options || {};
  const maxCellLength = options.maxCellLength ?? 200;
  const maxRowsPreview = options.maxRowsPreview ?? 100;
  const includeStyle = options.includeStyle ?? true;

  // If a single result object was passed (rows + fields), wrap it into a Response-like shape
  let response;
  if (
    responseOrResult &&
    typeof responseOrResult === 'object' &&
    Array.isArray(responseOrResult.rows) &&
    Array.isArray(responseOrResult.fields) &&
    !Array.isArray(responseOrResult.results)
  ) {
    response = {
      query: '',
      results: [responseOrResult],
    };
  } else {
    // assume it's already a Response-like object
    response = responseOrResult || { query: '', results: [] };
  }

  // Apply optional metadata from options (so renderReplHtml(ret, {query, text}) works)
  if (typeof options.query === 'string' && options.query.length > 0) {
    response.query = options.query;
  }
  if (typeof options.text === 'string' && options.text.length > 0) {
    response.text = options.text;
  }

  function escapeHtml(str) {
    return String(str)
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#39;');
  }

  function cellClass(value) {
    if (value === null) return 'PGliteRepl-null';
    if (typeof value === 'number') return 'PGliteRepl-number';
    if (typeof value === 'boolean') return 'PGliteRepl-boolean';
    return '';
  }

  function formatCellValue(value) {
    let str;
    if (value === null) {
      str = 'null';
    } else if (typeof value === 'number') {
      str = value.toString();
    } else if (typeof value === 'boolean') {
      str = value ? 'true' : 'false';
    } else if (value instanceof Date) {
      str = value.toISOString()
      str = str.replace('T00:00:00.000Z', '')
    } else if (Array.isArray(value)) {
      str = '[' + value.map(formatCellValue).join(', ') + ']';
    } else if (typeof value === 'object' && value !== null) {
      try {
        if (ArrayBuffer.isView(value)) {
          str = `${value.byteLength} bytes`;
        } else {
          str = JSON.stringify(value);
        }
      } catch (e) {
        str = String(value);
      }
    } else if (value === undefined) {
      str = 'undefined';
    } else {
      str = String(value);
    }
    if (str.length > maxCellLength) str = str.slice(0, maxCellLength) + '…';
    return str;
  }

  function rowToArray(row, fields) {
    if (Array.isArray(row)) return row;
    if (typeof row === 'object' && row !== null) {
      if (Array.isArray(fields) && fields.length > 0) {
        return fields.map((f) => row[f.name]);
      } else {
        return Object.values(row);
      }
    }
    return [row];
  }

  // Dark theme styles matching DuckDB template
  const style = `<style>
  .PGliteRepl-root {
    font-family: system-ui, -apple-system, Segoe UI, Roboto, sans-serif;
    color: #eee;
    background: transparent;
    line-height: 1.4;
    margin: 0;
    padding: 0;
    font-size: 13px;
  }

  .PGliteRepl-line {
    margin: 0.5em 0;
    white-space: pre-wrap;
  }

  .PGliteRepl-query {
    background: #2b2b2b;
    border: 1px solid #555;
    padding: 0.4em 0.6em;
    border-radius: 4px;
    color: #fff;
    overflow-x: auto;
    margin: 0;
    line-height: 1.4;
    font-family: inherit;
    font-weight: 600;
  }

  .PGliteRepl-text {
    color: #aaa;
    margin: 0.4em 0 0;
    font-style: italic;
  }

  .PGliteRepl-error {
    color: #ffd2d2;
    background: #2b1515;
    padding: 0.4em 0.6em;
    border-radius: 4px;
    border: 1px solid #553333;
    margin: 0;
  }

  .PGliteRepl-table-scroll {
    overflow: visible;
    max-width: 100%;
    margin: 0.5em 0;
    position: relative;
  }

  .PGliteRepl-table {
    border-collapse: collapse;
    width: 100%;
    font-family: inherit;
    color: #eee;
    background: #1e1e1e;
    border: 1px solid #444;
  }

  .PGliteRepl-table th,
  .PGliteRepl-table td {
    padding: 0.4em 0.6em;
    border: 1px solid #333;
    text-align: left;
    vertical-align: top;
  }

  .PGliteRepl-table thead th {
    position: sticky;
    top: 0;
    background: #2b2b2b;
    border: 1px solid #555;
    color: #fff;
    font-weight: 600;
    z-index: 10;
    box-shadow: 0 2px 4px rgba(0,0,0,0.3);
  }

  .PGliteRepl-col-num {
    width: 56px;
    text-align: right;
    color: #888;
    background: #262626;
    font-style: italic;
    position: sticky;
    left: 0;
    z-index: 11;
  }

  .PGliteRepl-rownum {
    width: 56px;
    text-align: right;
    color: #888;
    background: #262626;
    font-style: italic;
    position: sticky;
    left: 0;
    z-index: 5;
  }

  .PGliteRepl-table tbody tr:nth-child(even) {
    background: #262626;
  }

  .PGliteRepl-table tbody tr:nth-child(odd) {
    background: #1e1e1e;
  }

  .PGliteRepl-null {
    color: #aaa;
    font-style: italic;
  }

  .PGliteRepl-number {
    text-align: right;
    font-variant-numeric: tabular-nums;
  }

  .PGliteRepl-boolean {
    font-weight: 600;
  }

  .PGliteRepl-table-row-count {
    color: #aaa;
    margin-top: 0.4em;
    display: block;
    font-size: 12px;
  }

  .PGliteRepl-divider {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-top: 0.5em;
  }

  .PGliteRepl-divider hr {
    border: none;
    height: 1px;
    background: #444;
    flex: 1;
  }

  .PGliteRepl-time {
    color: #aaa;
    font-size: 12px;
  }

  @media (max-width: 480px) {
    .PGliteRepl-table th,
    .PGliteRepl-table td {
      padding: 0.3em 0.4em;
      font-size: 11px;
    }
    .PGliteRepl-query {
      font-size: 12px;
      padding: 0.3em 0.4em;
    }
    .PGliteRepl-col-num,
    .PGliteRepl-rownum {
      width: 40px;
    }
  }
  </style>`;

  const htmlParts = [];
  htmlParts.push(`<div class="PGliteRepl-root">`);
  if (includeStyle) htmlParts.push(style);

  if (response.query && response.query.length > 0) {
    htmlParts.push(
      `<pre class="PGliteRepl-line PGliteRepl-query">${escapeHtml(response.query)}</pre>`,
    );
  }

  if (response.text) {
    htmlParts.push(
      `<div class="PGliteRepl-line PGliteRepl-text">${escapeHtml(response.text)}</div>`,
    );
  }

  if (response.error) {
    htmlParts.push(
      `<div class="PGliteRepl-line PGliteRepl-error">${escapeHtml(response.error)}</div>`,
    );
  } else if (Array.isArray(response.results)) {
    response.results.forEach((result, resultIndex) => {
      if (!result.fields || result.fields.length === 0) {
        htmlParts.push(
          `<div class="PGliteRepl-line"><div class="PGliteRepl-null">ok</div></div>`,
        );
        return;
      }

      htmlParts.push(`<div class="PGliteRepl-line">`);
      htmlParts.push(`<div class="PGliteRepl-table-scroll"><table class="PGliteRepl-table">`);

      // headers: add row-number column first
      htmlParts.push('<thead><tr>');
      htmlParts.push(`<th class="PGliteRepl-col-num">#</th>`);
      result.fields.forEach((col) => {
        htmlParts.push(`<th>${escapeHtml(col.name)}</th>`);
      });
      htmlParts.push('</tr></thead>');

      // rows
      const rows = Array.isArray(result.rows) ? result.rows : [];
      const previewRows = rows.slice(0, maxRowsPreview);
      htmlParts.push('<tbody>');
      previewRows.forEach((row, rowIndex) => {
        const rowArr = rowToArray(row, result.fields);
        htmlParts.push('<tr>');
        // row number (1-based)
        const rowNumber = rowIndex + 1;
        htmlParts.push(`<td class="PGliteRepl-rownum">${escapeHtml(rowNumber)}</td>`);
        rowArr.forEach((col) => {
          const raw = formatCellValue(col);
          const cls = cellClass(col);
          htmlParts.push(`<td class="${cls}">${escapeHtml(raw)}</td>`);
        });
        htmlParts.push('</tr>');
      });
      htmlParts.push('</tbody>');

      htmlParts.push('</table></div>');

      const total = rows.length;
      if (total > maxRowsPreview) {
        htmlParts.push(
          `<div class="PGliteRepl-table-row-count">${maxRowsPreview} of ${total} rows — truncated</div>`,
        );
      } else {
        htmlParts.push(`<div class="PGliteRepl-table-row-count">${total} rows</div>`);
      }

      htmlParts.push('</div>'); // end PGliteRepl-line
    });
  }

  if (typeof response.time === 'number') {
    htmlParts.push(
      `<div class="PGliteRepl-divider"><hr style="flex:1;margin:0"/><div class="PGliteRepl-time">${Number(
        response.time,
      ).toFixed(1)}ms</div></div>`,
    );
  }

  htmlParts.push('</div>'); // .PGliteRepl-root
  return htmlParts.join('');
}
@end

@PGlite.eval
<script>
const id = "@0"

setTimeout(async () => {
  await window.accessQueue.grantAccess()
  const db = window.getDatabase(id)
  const statements = `@input`
  for (const stmt of statements.split(';')) {
    const trimmed = stmt.trim();
    if (trimmed.length === 0) continue;
    if (trimmed.toLowerCase().startsWith('erdiagram') ) {
        let diagram = await db.query(window.dbdiagramQuery);
        diagram = diagram.rows.map(r => r["dbdiagram"]).join("\n\n");
        window.console.log(diagram);
        console.html(window.dbdiagram(diagram));
        continue;
    }
    try {
      let result = await db.query(trimmed);
      console.html(renderReplHtml(result, { query: trimmed, ret: result }) );
    } catch (e) {
      console.html(renderReplHtml({ query: trimmed, error: e.message }));
      break;
    }
  }

  window.accessQueue.releaseAccess()

  send.lia('')
}, 100)

"LIA: wait"
</script>
@end

@PGlite.terminal
<script>
const id = "@0"

const statements = `@input`
setTimeout(async () => {
  await window.accessQueue.grantAccess()
  const db = window.getDatabase(id)
  for (const stmt of statements.split(';')) {
    const trimmed = stmt.trim();
    try {
        if (trimmed.length === 0) continue;
        if (trimmed.toLowerCase().startsWith('erdiagram') ) {
          let diagram = await db.query(window.dbdiagramQuery);
          diagram = diagram.rows.map(r => r["dbdiagram"]).join("\n\n");

          console.html(window.dbdiagram(diagram));
          continue;
       }
      
        let result = await db.query(trimmed);
        console.html(renderReplHtml(result, { query: trimmed, ret: result }) );
    } catch (e) {
        console.html(renderReplHtml({ query: trimmed, error: e.message }));
        break;
    }
  }

  window.accessQueue.releaseAccess()

  send.handle("input", async (input) => {
    // Run a query
    const statements = input
        .split(';')
        .map(s => s.trim())
        .filter(s => s.length > 0);

    for (const stmt of statements) {
        try {
            if (stmt.length === 0) continue;
            if (stmt.toLowerCase().startsWith('erdiagram') ) {
                let diagram = await db.query(window.dbdiagramQuery);
                diagram = diagram.rows.map(r => r["dbdiagram"]).join("\n\n");
                console.html(window.dbdiagram(diagram));
                continue;
            }
            let result = await db.query(stmt);
            console.html(renderReplHtml(result, { query: '', ret: result }) );
        } catch (e) {
            console.html(renderReplHtml({ query: '', error: e.message }));
            break;
        }
    }
  });
}, 100)

"LIA: terminal"
</script>
@end

@PGlite.js
<script>
setTimeout(async () => {
await window.accessQueue.grantAccess()
const db = window.getDatabase("@0")
try {
@input
} catch (e) {
    console.error(e.message);
}

window.accessQueue.releaseAccess()

send.lia('')
}, 100)

"LIA: wait"
</script>
@end
-->

# PGlite

                         --{{0}}--
PGlite is a lightweight WASM Postgres build packaged into a TypeScript library
that runs entirely in the browser. This LiaScript template allows you to embed
interactive PostgreSQL queries directly in your educational materials, enabling
students to experiment with SQL and database operations in the browser using
PGlite.

For more information about PGlite, visit the [PGlite website](https://pglite.dev).

**Try it on LiaScript:**

https://liascript.github.io/course/?https://raw.githubusercontent.com/LiaTemplates/PGlite/main/README.md

**See the project on Github:**

https://github.com/LiaTemplates/PGlite

**Experiment in the LiveEditor:**

https://liascript.github.io/LiveEditor/?/show/file/https://raw.githubusercontent.com/LiaTemplates/PGlite/main/README.md

                         --{{1}}--
Like with other LiaScript templates, there are three ways to integrate PGlite,
but the easiest way is to copy the import statement into your project.

                           {{1}}
1. Load the latest macros via (this might cause breaking changes):

   `import: https://raw.githubusercontent.com/LiaTemplates/PGlite/main/README.md`

   or the current version 0.0.4 via:

   `import: https://raw.githubusercontent.com/LiaTemplates/PGlite/0.0.4/README.md`

2. Copy the definitions into your Project

3. Clone this repository on GitHub

## `@PGlite.eval`

                         --{{0}}--
This is the most common way to run PostgreSQL queries in LiaScript. It executes
SQL queries and displays the results in a nicely formatted table. The macro
requires a database name parameter (which allows you to have multiple independent
databases) and executes the SQL code block.

```sql
SELECT 'Hello, PGlite!' AS greeting, 42 AS answer;
```
@PGlite.eval(demo)

                         --{{1}}--
You can create tables, insert data, and perform complex queries:

                           {{1}}
```sql
-- Create a table with sample data
CREATE TABLE weather (
    city            VARCHAR(80),
    temp_lo         INT,
    temp_hi         INT,
    prcp            REAL,
    date            DATE
);

INSERT INTO weather VALUES ('San Francisco', 46, 50, 0.25, '1994-11-27');
INSERT INTO weather VALUES ('San Francisco', 43, 57, 0.0, '1994-11-29');
INSERT INTO weather VALUES ('Hayward', 37, 54, NULL, '1994-11-29');

-- Query the data
SELECT city, (temp_hi + temp_lo) / 2 AS temp_avg, date
FROM weather
ORDER BY temp_avg DESC;
```
@PGlite.eval(demo)

                         --{{2}}--
PGlite supports full PostgreSQL functionality including aggregations, joins, and
window functions:

                           {{2}}
```sql
SELECT 
    city,
    COUNT(*) AS num_readings,
    AVG(temp_hi) AS avg_high,
    MAX(temp_hi) AS max_high,
    MIN(temp_lo) AS min_low
FROM weather
GROUP BY city
ORDER BY city;
```
@PGlite.eval(demo)

## `@PGlite.terminal`

                         --{{0}}--
This macro creates an interactive terminal where users can execute multiple
queries in succession. Unlike `@PGlite.eval`, which executes once, the terminal
mode allows continuous interaction with the database. Users can type queries and
see results immediately.

```sql
-- Initial query to set up the database
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2),
    category VARCHAR(50)
);

INSERT INTO products (name, price, category) VALUES
    ('Laptop', 999.99, 'Electronics'),
    ('Mouse', 29.99, 'Electronics'),
    ('Desk', 299.99, 'Furniture'),
    ('Chair', 199.99, 'Furniture');

-- Try running these queries in the terminal below:
-- SELECT * FROM products;
-- SELECT category, AVG(price) AS avg_price FROM products GROUP BY category;
-- SELECT * FROM products WHERE price > 100 ORDER BY price;
```
@PGlite.terminal(shop)

                         --{{1}}--
The terminal maintains the database state, so you can build upon previous
queries. This is excellent for teaching incremental database operations and
allowing students to explore data interactively.

## `@PGlite.js`

                         --{{0}}--
For advanced use cases, this macro allows you to write custom JavaScript code
that interacts with the PGlite database object. This is useful when you need
more control over query execution, want to process results programmatically, or
integrate with other JavaScript libraries.

```js
// Create a table with sample data
await db.exec(`
  CREATE TABLE sales (
      month VARCHAR(10),
      product VARCHAR(20),
      revenue INTEGER
  );
  
  INSERT INTO sales VALUES
      ('2024-01', 'Product A', 1500),
      ('2024-01', 'Product B', 2300),
      ('2024-02', 'Product A', 1800),
      ('2024-02', 'Product B', 2100),
      ('2024-03', 'Product A', 2200),
      ('2024-03', 'Product B', 2500);
`);

// Query and process results
const result = await db.query(`
  SELECT 
    product,
    SUM(revenue) AS total_revenue,
    AVG(revenue)::INTEGER AS avg_revenue,
    COUNT(*) AS num_months
  FROM sales
  GROUP BY product
  ORDER BY total_revenue DESC
`);

// Access the results
const data = result.rows;
console.log("Sales Analysis:");

// Display results in a custom format
let output = '<div style="padding: 10px; background: #1e1e1e; color: #eee; font-family: monospace;">';
output += '<h3 style="color: #fff; margin-top: 0;">Sales Summary</h3>';

for (const row of data) {
  output += `<div style="margin: 8px 0; padding: 8px; background: #2b2b2b; border-left: 3px solid #4a9eff;">`;
  output += `<strong style="color: #4a9eff;">${row.product}</strong><br/>`;
  output += `Total Revenue: $${row.total_revenue.toLocaleString()}<br/>`;
  output += `Average: $${row.avg_revenue.toLocaleString()}<br/>`;
  output += `Months: ${row.num_months}`;
  output += `</div>`;
}

output += '</div>';
console.html(output);
```
@PGlite.js(analytics)

                         --{{1}}--
The database object (`db`) is automatically provided and connected. You have
full access to the PGlite JavaScript API, allowing you to execute complex
workflows, handle errors, and integrate with other browser APIs.

                           {{1}}
> **Note on numeric values:** PostgreSQL often returns numeric aggregations with
> high precision. To avoid display issues, cast large numbers to INTEGER using
> `::INTEGER` in your SQL queries, or handle numeric values explicitly in your
> JavaScript code using `.toString()` or `Number()` conversion.

## Advanced Features

                         --{{0}}--
PGlite supports the full PostgreSQL feature set including window functions, CTEs
(Common Table Expressions), and complex aggregations:

```sql
-- Create sample time-series data
CREATE TABLE daily_sales (
    sale_date DATE,
    sales INTEGER,
    store VARCHAR(20)
);

INSERT INTO daily_sales
SELECT 
    DATE '2024-01-01' + (day || ' days')::INTERVAL AS sale_date,
    50 + FLOOR(RANDOM() * 50) AS sales,
    'Store ' || ((day % 3) + 1) AS store
FROM generate_series(0, 29) AS day;

-- Calculate moving averages with window functions
SELECT 
    sale_date,
    store,
    sales,
    AVG(sales) OVER (
        PARTITION BY store 
        ORDER BY sale_date 
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3day
FROM daily_sales
ORDER BY store, sale_date
LIMIT 15;
```
@PGlite.eval(advanced)

                         --{{1}}--
You can also use CTEs for complex queries:

                           {{1}}
```sql
-- Using Common Table Expressions (CTEs)
WITH weekly_stats AS (
    SELECT 
        store,
        DATE_TRUNC('week', sale_date) AS week,
        SUM(sales) AS weekly_total,
        AVG(sales) AS weekly_avg
    FROM daily_sales
    GROUP BY store, DATE_TRUNC('week', sale_date)
)
SELECT 
    store,
    COUNT(*) AS num_weeks,
    AVG(weekly_total)::INTEGER AS avg_weekly_sales,
    MAX(weekly_total) AS best_week,
    MIN(weekly_total) AS worst_week
FROM weekly_stats
GROUP BY store
ORDER BY avg_weekly_sales DESC;
```
@PGlite.eval(advanced)

## Database Visualization

                         --{{0}}--
PGlite includes a special feature to visualize your database schema using
dbdiagram.io. Simply run the `erdiagram` command to generate an interactive
entity-relationship diagram of all your tables.

```sql
-- Create multiple related tables
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    order_date DATE,
    total DECIMAL(10,2)
);

CREATE TABLE order_items (
    item_id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(order_id),
    product_name VARCHAR(100),
    quantity INTEGER,
    price DECIMAL(10,2)
);

ERDIAGRAM; -- Visualize the schema
```
@PGlite.eval(visualization)

## Database Isolation

                         --{{0}}--
Each macro call can use a different database name (the parameter in
parentheses). This allows you to have multiple independent databases in the same
document:

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);

INSERT INTO users (name, email) VALUES
    ('Alice', 'alice@example.com'),
    ('Bob', 'bob@example.com');

SELECT * FROM users;
```
@PGlite.eval(db1)

```sql
-- This is a completely separate database
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    description VARCHAR(200),
    amount DECIMAL(10,2)
);

INSERT INTO orders (description, amount) VALUES
    ('Order A', 150.00),
    ('Order B', 200.00);

SELECT * FROM orders;
```
@PGlite.eval(db2)

                         --{{1}}--
The database name parameter ensures isolation between different examples and
exercises in your course material.

## PostgreSQL Extensions

                         --{{0}}--
PGlite includes support for pgTAP, a unit testing framework for PostgreSQL. This
allows you to write and run database tests directly in your educational
materials.

```sql
-- Create a test suite
CREATE EXTENSION IF NOT EXISTS pgtap;

-- Test table creation
SELECT plan(3);

CREATE TABLE test_table (
    id SERIAL PRIMARY KEY,
    value INTEGER
);

SELECT has_table('test_table', 'test_table should exist');
SELECT has_column('test_table', 'id', 'test_table should have id column');
SELECT has_column('test_table', 'value', 'test_table should have value column');

SELECT * FROM finish();
```
@PGlite.eval(testing)

## Use Cases

                         --{{0}}--
This template is perfect for:

* **Teaching SQL**: Interactive SQL tutorials where students can modify and run queries
* **Database Design Courses**: Demonstrate table creation, constraints, and relationships
* **Database Concepts**: Show transactions, indexes, and query optimization
* **Application Development**: Teach how to interact with databases from code
* **Data Analysis**: Pre-process and explore datasets with SQL
* **Testing Practices**: Demonstrate database testing with pgTAP
* **Interactive Examples**: Allow readers to experiment with queries in documentation

## Implementation

                         --{{0}}--
The LiaScript implementation of PGlite is based on @electric-sql/pglite, which
runs entirely in the browser using WebAssembly. The implementation includes
custom table rendering with a dark theme and support for multiple concurrent
database instances.

### Technical Details

* **PGlite WASM**: Uses the official `@electric-sql/pglite` package
* **PostgreSQL Compatible**: Full PostgreSQL 16 feature set
* **Table Rendering**: Custom HTML table renderer with dark theme styling
* **Multiple Databases**: Supports concurrent isolated database instances
* **Result Display**: Automatic formatting of query results with row limiting
* **Extensions**: Includes pgTAP extension for database testing
* **Schema Visualization**: Integration with dbdiagram.io for ER diagrams

## Examples

                         --{{0}}--
For more examples and detailed PostgreSQL usage, see the [PostgreSQL documentation](https://www.postgresql.org/docs/)

---

                         --{{0}}--
Here's a complete example combining multiple features:

```sql
-- Complete example: Student grade tracking system
CREATE TABLE students (
    student_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);

CREATE TABLE courses (
    course_id SERIAL PRIMARY KEY,
    course_name VARCHAR(100),
    credits INTEGER
);

CREATE TABLE enrollments (
    enrollment_id SERIAL PRIMARY KEY,
    student_id INTEGER REFERENCES students(student_id),
    course_id INTEGER REFERENCES courses(course_id),
    grade DECIMAL(3,2),
    semester VARCHAR(20)
);

-- Insert sample data
INSERT INTO students (name, email) VALUES
    ('Alice Johnson', 'alice@university.edu'),
    ('Bob Smith', 'bob@university.edu'),
    ('Carol White', 'carol@university.edu');

INSERT INTO courses (course_name, credits) VALUES
    ('Database Systems', 4),
    ('Web Development', 3),
    ('Data Structures', 4);

INSERT INTO enrollments (student_id, course_id, grade, semester) VALUES
    (1, 1, 3.7, 'Fall 2024'),
    (1, 2, 4.0, 'Fall 2024'),
    (2, 1, 3.3, 'Fall 2024'),
    (2, 3, 3.8, 'Fall 2024'),
    (3, 2, 3.9, 'Fall 2024'),
    (3, 3, 4.0, 'Fall 2024');

-- Complex query with joins and aggregations
SELECT 
    s.name,
    COUNT(e.course_id) AS courses_taken,
    AVG(e.grade)::DECIMAL(3,2) AS gpa,
    SUM(c.credits) AS total_credits
FROM students s
JOIN enrollments e ON s.student_id = e.student_id
JOIN courses c ON e.course_id = c.course_id
GROUP BY s.student_id, s.name
ORDER BY gpa DESC;
```
@PGlite.eval(complete_example)

## License

                         --{{0}}--
This template is released under the CC0-1.0 license, meaning you can use it
freely in your projects without restrictions.



