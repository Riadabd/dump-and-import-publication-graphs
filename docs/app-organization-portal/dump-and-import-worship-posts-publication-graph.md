# Worship Posts

This document assumes that `triplestore` and `publication-triplestore` production data are loaded in your local repository docker containers.

With production data, it's better to add the following to your `docker-compose.override.yml` so that the databases use their production configurations:

```yaml
triplestore:
  volumes:
    - ./config/triplestore/virtuoso-production.ini:/data/virtuoso.ini

publication-triplestore:
  volumes:
    - ./config/publication-triplestore/virtuoso-production.ini:/data/virtuoso.ini
```

## Backups

If you are running this procedure on a work server, make sure to backup `triplestore` and `publication-triplestore`.

### Triplestore

```sh
/data/useful-scripts/virtuoso-backup.sh `docker ps --filter "label=com.docker.compose.project=[YOUR-PROJECT]" --filter "label=com.docker.compose.service=triplestore" --format "{{.Names}}"`
```

Depending on the server you may be on, `virtuoso-backup.sh` may be called `virtuoso_backup.sh`; confirm by checking the `/data/useful-scripts` folder.

`[YOUR-PROJECT]` should be replaced by your intended application. In the case of `app-organization-portal`:
* `PROD`: `app-organization-portal`

### Publication-triplestore

```sh
/data/useful-scripts/virtuoso-backup.sh `docker ps --filter "label=com.docker.compose.project=[YOUR-PROJECT]" --filter "label=com.docker.compose.service=publication-triplestore" --format "{{.Names}}"`
```

Depending on the server you may be on, `virtuoso-backup.sh` may be called `virtuoso_backup.sh`; confirm by checking the `/data/useful-scripts` folder.

`[YOUR-PROJECT]` should be replaced by your intended application. In the case of `app-organization-portal`:
* `PROD`: `app-organization-portal`

## Count Check

```sparql
SELECT COUNT(*) WHERE {
  GRAPH <http://redpencil.data.gift/id/deltas/producer/worship-posts> {
    ?s ?p ?o .
  }
}
```

Run the above query on [http://localhost:8890/sparql](http://localhost:8890/sparql) (for `triplestore`) and [http://localhost:8891/sparql](http://localhost:8891/sparql) for `publication-triplestore`. It should output a value on `triplestore` and 0 on `publication-triplestore`.

## Dump the Named Graph

We first need to dump the graph using the `dump_one_graph` procedure: [https://vos.openlinksw.com/owiki/wiki/VOS/VirtRDFDatasetDump#Dump%20One%20Graph](https://vos.openlinksw.com/owiki/wiki/VOS/VirtRDFDatasetDump#Dump%20One%20Graph).

It can be used to dump any single named graph:

```
CREATE PROCEDURE dump_one_graph
  ( IN  srcgraph           VARCHAR
  , IN  out_file           VARCHAR
  , IN  file_length_limit  INTEGER  := 1000000000
  )
  {
    DECLARE  file_name     VARCHAR;
    DECLARE  env,  ses           ANY;
    DECLARE  ses_len
          ,  max_ses_len
          ,  file_len
          ,  file_idx      INTEGER;
   SET ISOLATION = 'uncommitted';
   max_ses_len  := 10000000;
   file_len     := 0;
   file_idx     := 1;
   file_name    := sprintf ('%s%06d.ttl', out_file, file_idx);
   string_to_file ( file_name || '.graph',
                     srcgraph,
                     -2
                   );
    string_to_file ( file_name,
                     sprintf ( '# Dump of graph <%s>, as of %s\n@base <> .\n',
                               srcgraph,
                               CAST (NOW() AS VARCHAR)
                             ),
                     -2
                   );
   env := vector (dict_new (16000), 0, '', '', '', 0, 0, 0, 0, 0);
   ses := string_output ();
   FOR (SELECT * FROM ( SPARQL DEFINE input:storage ""
                         SELECT ?s ?p ?o { GRAPH `iri(?:srcgraph)` { ?s ?p ?o } }
                       ) AS sub OPTION (LOOP)) DO
      {
        http_ttl_triple (env, "s", "p", "o", ses);
        ses_len := length (ses);
        IF (ses_len > max_ses_len)
          {
            file_len := file_len + ses_len;
            IF (file_len > file_length_limit)
              {
                http (' .\n', ses);
                string_to_file (file_name, ses, -1);
                gz_compress_file (file_name, file_name||'.gz');
                file_delete (file_name);
                file_len := 0;
                file_idx := file_idx + 1;
                file_name := sprintf ('%s%06d.ttl', out_file, file_idx);
                string_to_file ( file_name,
                                 sprintf ( '# Dump of graph <%s>, as of %s (part %d)\n@base <> .\n',
                                           srcgraph,
                                           CAST (NOW() AS VARCHAR),
                                           file_idx),
                                 -2
                               );
                 env := VECTOR (dict_new (16000), 0, '', '', '', 0, 0, 0, 0, 0);
              }
            ELSE
              string_to_file (file_name, ses, -1);
            ses := string_output ();
          }
      }
    IF (LENGTH (ses))
      {
        http (' .\n', ses);
        string_to_file (file_name, ses, -1);
        gz_compress_file (file_name, file_name||'.gz');
        file_delete (file_name);
      }
  }
;
```

Copy and paste the above procedure inside the `isql-v` triplestore interface (`docker compose exec triplestore isql-v`) and run it; this will load the procedure and allow you to call it. Create a new folder inside `app-organization-portal` (`mkdir data/db/worship_posts_producer_delta_producer_graph_dump`) and run this command:

```
SQL> dump_one_graph ('http://redpencil.data.gift/id/deltas/producer/worship-posts', './worship_posts_producer_delta_producer_graph_dump/data_', 1000000000);

SQL> exec('checkpoint');
```

This will dump the `<http://redpencil.data.gift/id/deltas/producer/worship-posts>` graph into `data_XXX.ttl.gz` and `data_XXX.ttl.graph` files (located in your `data/db/worship_posts_producer_delta_producer_graph_dump/` folder):

```
$ ls
data_000001.ttl.gz
data_000002.ttl.gz
....
data_000001.ttl.graph
data_000002.ttl.graph
```

Make sure you unzip the `.gz` files.

* Unzip and remove original:

```sh
gzip -d <.gz>
```

* Unzip and keep original:

```sh
gzip -dk <.gz>
```

## Import into Publication-Triplestore

The goal is to import the dumped data into `publication-triplestore`.

Add the below snippet into your `docker-compose.override.yml` file; note the second `volumes` entry that maps the dumped data from the previous section into `/tmp/worship_posts_producer_delta_producer_graph_dump/`.

```yaml
publication-triplestore:
  volumes:
    - ./config/publication-triple-store/virtuoso-production.ini:/data/virtuoso.ini
    - /path/to/data/db/worship_posts_producer_delta_producer_graph_dump/:/tmp/worship_posts_producer_delta_producer_graph_dump
```

Run `docker compose up -d publication-triplestore && docker compose exec publication-triplestore isql-v` and execute the below command in the `isql-v` interface to import the files (replace xxx by the `.ttl` files you find inside); follow it up by running a checkpoint. In the case of `worship-posts`, there will most likely be only one file (`/tmp/worship_posts_producer_delta_producer_graph_dump/data_000001.ttl`).

```
DB.DBA.TTLP_MT(file_to_string_output('/tmp/worship_posts_producer_delta_producer_graph_dump/data_xxx.ttl'), '', 'http://redpencil.data.gift/id/deltas/producer/worship-posts');
```

```
exec('checkpoint');
```

### Count Check

```sparql
SELECT COUNT(*) WHERE {
  GRAPH <http://redpencil.data.gift/id/deltas/producer/loket-leidinggevenden-producer> {
    ?s ?p ?o .
  }
}
```

Run the above query on [http://localhost:8890/sparql](http://localhost:8890/sparql) (for `virtuoso`) and [http://localhost:8891/sparql](http://localhost:8891/sparql) for `publication-triplestore`. It should output the same value on `virtuoso` and `publication-triplestore`.

## Delete Graph from Virtuoso

After importing the data into `publication-triplestore`, delete the graph and all its entries from `virtuoso` by running the following query inside the `virtuoso` SPARQL endpoint (`localhost:8890/sparql` by default):

```sparql
CLEAR GRAPH <http://redpencil.data.gift/id/deltas/producer/worship-posts>
```

Enter `triplestore`'s `isql-v` interface again (`docker compose exec triplestore isql-v`) and execute a checkpoint:

```
exec('checkpoint');
```

### Count Check

```sparql
SELECT COUNT(*) WHERE {
  GRAPH <http://redpencil.data.gift/id/deltas/producer/worship-posts> {
    ?s ?p ?o .
  }
}
```

Run the above query on [http://localhost:8890/sparql](http://localhost:8890/sparql) for `triplestore` and [http://localhost:8891/sparql](http://localhost:8891/sparql) for `publication-triplestore`. It should output 0 on `triplestore` and the previous `virtuoso` value on `publication-triplestore`.
