<script lang="ts">
  import { initDB } from '$lib/duckdb'
  import Speech from '$lib/components/speech.svelte'
  import Spinner from '$lib/components/Spinner.svelte'
  export let data;
  import { base } from '$app/paths';
  
  const db = await initDB();
  
  async function load_db() {
    console.log("LOADING DB")
    // A simple case of a db of a single parquet file.
    //const db = await initDB();
    await db.registerFileURL('SOTU.parquet', `${base}/SOTU.parquet`, 4, false);
    const conn = await db.connect();

    console.log("PREPARING INSERT")
    await conn.query(`CREATE TABLE p1 AS SELECT * FROM parquet_scan('SOTU.parquet')`);
    await conn.query(`CREATE VIEW wordcounts_raw AS SELECT * FROM (SELECT "@id" id, 
        UNNEST("nc:unigrams").word0 as word, 
        UNNEST("nc:unigrams").count as count FROM
        p1) t1
    `);
    await conn.query(`
      CREATE TABLE wordcounts AS SELECT * FROM 
      wordcounts_raw
      NATURAL JOIN (SELECT word, SUM(count) as tot, COUNT(*) AS df FROM wordcounts_raw GROUP BY word) t2
    `);
    return conn;
  }

  // Set up the db connection as an empty promise.
  const conn_prom = load_db();

  const years = []
  // We could get this from the database, but no need to show off.
  for (let y = 1934; y < 2021; y++) {
    years.push(y)
  }


  $: results = new Promise(() => {})


  async function get_year(y) {
//    year = y;
    const conn = await conn_prom;
    results = conn.query(`
      SELECT * FROM parquet_scan('SOTU.parquet') WHERE year == ${y}
    `)
    const result = await results;
    console.log({results, result})
  }

  // Start with FDR, because he's good.
  conn_prom.then(() => get_year(1936))

    // T - Adding fileupload, 
    let files;

let fileContent = '';

const handleFileUpload = () => {
  const file = files[0];

  if (!file) {
    return;
  }

  const reader = new FileReader();

  reader.onload = (e) => {
    fileContent = e.target.result;
    // Use the content of the file here
    //console.log(fileContent);

    // insert into database
    console.log("INSERT CSV")
    insertCSV(fileContent).catch((err) => {
      console.error('Error inserting CSV file into DuckDB: ', err);
    });
  };

  reader.readAsText(file);
};



async function insertCSV(csv_data) {
    
    const data = csv_data;

    const conn = await conn_prom;

    await db.registerFileText(`data.csv`, data);

    await db.insertCSVFromPath( 'data.csv', {
        schema: 'main',
        name: 'foo',
        detect: true,
        header: true

    });


    console.log('CSV file inserted into DuckDB successfully.');
}







</script>


<h1>Duck DB Svelte-kit demo</h1>

<h2>Upload a csv file:</h2>
<input accept=".csv" bind:files id="avatar" name="avatar" type="file" on:change={handleFileUpload} />

{#if files}
<h2>File selected:</h2>
<p>{files[0].name} ({files[0].size} bytes)</p> 
<p>{fileContent}</p>



{/if}


<div class="description">
  This is a demo of a static, database-driven page using duckdb WASM.
  All the primary text is stored in a parquet file structured to the <a href="https://nonconsumptive.org">
  nonconsumptive</a> text sharing spec. On load, it gets a copy of duckdb WASM and uses it to mount 
  the parquet file into the database. Clicking on any year will load the state of the Union address for that year.
</div>

<div class="years">
  {#each years as y}
    <div class="year" on:click={() => get_year(y)}>{y}</div>
  {/each}
</div>
{#await results}
<Spinner></Spinner>
{:then result}
<Speech {results} />
{/await}
<style>
.years {
  display: flex;
  font-size: .75em;
  flex-wrap: wrap;
  flex-direction: row;
  justify-content: center;
  width: 90vw;
}
.year {
  cursor: pointer;
  padding: 0.1rem;
  border: 1px solid black;
  margin: 0.1rem;
}
.year:hover {
  background: lightgray;
}
</style>