Example project showing the usage of Elastic Enterprise Search [App Search](https://www.elastic.co/app-search/) with a [Search-UI](https://github.com/elastic/search-ui) React frontend. 

```
Download
https://upload.elastic.co/d/e37df6c75772a4bbce622d6af121d2a94902c09fc6af39bd1a90728b1be12fbb with token 09ab046498f56113
curl -L -H 'Authorization: 09ab046498f56113' -o 'app_search_scbcn2021.tar.bz2' https://upload.elastic.co/d/e37df6c75772a4bbce622d6af121d2a94902c09fc6af39bd1a90728b1be12fbb
```

All features of the Elastic stack that are presented here are [available for free](https://www.elastic.co/subscriptions).


## Prerequisites
* `Docker` and `docker-compose` (> 1.27) installed on your host
* [Search-UI](https://github.com/elastic/search-ui) requires [npm](https://www.npmjs.com/).](* Data transformation require various tools, [`csv2json`](https://github.com/darwin/csv2json), which relies on `orderhash` as a [missing dependency](https://github.com/darwin/csv2json/issues/12)
* MacOS dependencies can be installed with [`brew`](https://brew.sh/)
* Ubuntu Linux hosts might need increase of virtual [memory map areas](https://stackoverflow.com/questions/11683850/how-much-memory-could-vm-use)
* Node script [`app-search-cli`](https://gist.github.com/JasonStoltz/41b29e71310743e94275fc02fac33095) to bulk upload JSON files to App Search from my colleague Jason, found on our [forums](https://discuss.elastic.co/t/bulk-upload-file-data-into-app-search-from-a-json-file/264640/5). Download, add executable permissions and use as  
`app-search-cli upload ~/Documents/pokemon.json http://localhost:3002 $engine $private-key`

<details>
  <summary>Command reference</summary>
  
  
    # Mac OS
    brew install yarn truncate

    # Linux
    sudo sysctl -w vm.max_map_count=65535
    sudo apt install npm #??node-d3-dsv
    npm install iconv

    # Install csv2json and dependency
    sudo gem install orderedhash # Without it we would see "cannot load such file -- orderedhash (LoadError)"
    sudo sudo gem install csv2json
</details>

## Start up the Elastic stack

* Download the workshop materials from ela.st/scbcn2021 and extract the `app_search_scbcn2021.tar.gz` file and enter the directory.
```
tar xvf app_search_scbcn2021.tar.gz
cd app_search_scbcn2021
```

* Get the Docker environment up and have the Docker images for Elasticsearch, Kibana, Enterprise Search, Metricbeat, etc. loaded.
```
./runAll.sh
```
You can check the progress using `docker ps` to see running containers and `docker logs -f $container`, e.g. for `es0`, `kibana0`, `enterprise-search0`. These three, plus `mon_metricbeat0` will be the main containers we need running.

Depending on the available disk space the [default watermarks](https://www.elastic.co/guide/en/elasticsearch/reference/7.15/modules-cluster.html#disk-based-shard-allocation) might be a bit too aggressive and we should redefine them:
```
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.low": "5gb",
    "cluster.routing.allocation.disk.watermark.high": "3gb",
    "cluster.routing.allocation.disk.watermark.flood_stage": "1gb",
    "cluster.info.update.interval": "1m"
  }
}
```

## Input data
We will be using open data provided by the Generalitat de Catalunya and looking at energy efficiency certificates for buildings in the region:
[Certificats d’eficiència energètica d’edificis](https://analisi.transparenciacatalunya.cat/es/Energia/Certificats-d-efici-ncia-energ-tica-d-edificis/j6ii-t3w2) which [development examples available](https://dev.socrata.com/foundry/analisi.transparenciacatalunya.cat/j6ii-t3w2).

### Data preparation
App Search has some requirements for its input data that we need to match. Field names need to
- be in lower case
- contain no whitespaces
- have no accents or special characters

<details>
  <summary>Detailed steps</summary>
  
  #### Retrieve and prepare input data
  1. Example with the CSV version of the [Certificats d’eficiència energètica d’edificis](https://analisi.transparenciacatalunya.cat/es/Energia/Certificats-d-efici-ncia-energ-tica-d-edificis/j6ii-t3w2):  
`wget -O ceee.csv "https://analisi.transparenciacatalunya.cat/api/views/j6ii-t3w2/rows.csv?accessType=DOWNLOAD&bom=true&format=true&sorting=true"`
  
  2. Bring the header in a matching format  
  Remove special characters | to lower case | replace whitespace with underscore | remove single quotes  
  `head -n1 ceee.csv | iconv -c -f utf8 -t ascii//TRANSLIT |   tr '[:upper:]' '[:lower:]' |  sed -e 's/ /_/g' | sed -e "s/'//g" > fixed_header`
  Should the output contain ``georefer`encia`` or similar, please see the troubleshooting seciton.
  
  3. Grab a subset of the data, we want to include records with geopoint data which are not found in the first thousands/
  `sed "1s/.*/$(cat fixed_header)/" ceee.csv | head -n 15000 | csv2json --pretty > ceee_15k.json`  
  
  4. Convert Geo data into the matching format from its [`Point` datatype](https://dev.socrata.com/docs/datatypes/point.html) that uses `"latitude, longitude"` to the [well-known-text](https://docs.opengeospatial.org/is/12-063r5/12-063r5.html) `"POINT(lon lat)"` data type [used by App Search](https://www.elastic.co/guide/en/app-search/7.15/api-reference.html#overview-api-references-geolocation).
  !! Depending on the system and shell used you will most probably need to change the `printf`/`echo` statement. 

  ```bash
  IFS='' cat ceee_15k.json |
  while read -r data || [[ -n "$data" ]]
  do
    if [[ $data == *"POINT"* ]];
    then
      echo $data | awk -F '[( )]' '{print "    \"georeferencia\": \"POINT (" $(NF-1) " " $(NF-2) ")\"" }'
    else
     printf '%s\n' $data # Mac OS
     # echo $data #  Linux
    fi
  done > ceee_15k_geo.json
  ```
</details>  

### Indexing the data

Data can be ingested into App Search in a [variety of ways](https://www.elastic.co/guide/en/app-search/7.15/indexing-documents-guide.html), we will use the API approach.

You can just download the prepared subset of records that has been converted into a compatible JSON file and upload one of the sample datasets with `15k`, `30k`, `50k` or `200k` documents.

* Upload the file to your engine  
  `app-search-cli upload ceee_15k_geo.json http://localhost:3002 test private-7i4iwuayhgvnapbp5quc8y4q`

* Create different engines for each dataset size and ingest the data accordingly. Check with in the `Overview` or `Documents` tabs in App Search to confirm if all documents have been uploaded and look at the API logs and usage metrics. Double check the Elasticsearch API output for number of documents and storage size using the API, e.g. via the [Kibana Dev Console](https://www.elastic.co/guide/en/kibana/7.15/console-kibana.html) using the [`_cat/indices` API](https://www.elastic.co/guide/en/elasticsearch/reference/7.15/cat-indices.html): `GET _cat/indices/*engine*?s=store.size:desc,index:asc&v=true`

#### Schema

Let us try to map some of the data fields to a particular type. You can choose any field, good candidates are for example `codi_postal`, `georeferencia` or `emissions_de_co2`. Depending on the size of the dataset you are using there might be documents that do not conform to the expected data type.

* Try changing the type for the various datasets and observe if all the fields contents can be parsed successfully.

## Building the Search UI

We use the App Search Engine tab `Search UI` to prepare a basic [Search UI](https://github.com/elastic/search-ui) interface for our use case, including filters, sort fields, etc. Download the prepared ZIP package to edit the view on your local machine.

After downloading the React application you will need to
```
Unzip
npm update 
npm run build
npm start
```

This will launch a local server and your browser to the search interface.

The structure of your example should look like the following:
```
src
├── App.js
├── _tests_
│   └── App.test.js
├── config
│   ├── config-helper.js
│   ├── engine.json
│   └── engine.json.example
└── index.js
```

By editing the `App.js` and `config-helper.js` file we can make changes to the pre-configured UI.

Look at the provided example code and compare the options chosen for the various item such as [facets](https://github.com/elastic/search-ui/blob/master/ADVANCED.md#facet), [paging](https://github.com/elastic/search-ui/blob/master/ADVANCED.md#paging0), 


## Troubleshooting
* Converting special characters with `iconv` behaves differently on MacOS (https://stackoverflow.com/questions/806199/how-to-fix-weird-issue-with-iconv-on-mac-os-x) and Ubuntu Linux this much cleaner. Running `iconv -f utf8 -t ascii//TRANSLIT $field_names` removes the special characters. Alternatively use `unaccent` or similar.
* After running through `csv2json` there might be a control sequence left over in the last line of the file. If the JSON upload fails this could be a reason. Sometimes we are also missing the closing KSON bracket `]` and should add it `SyntaxError: Unexpected token C in JSON at position 542707`

## Observations, tips
- App Search is strict on parsing all records fields according to its [data schema](https://www.elastic.co/guide/en/app-search/7.15/api-reference.html). Fields that are not populated need to be `null` as `""` for example would lead to a parsing issues. We can try to change the mapping of the fields `codi_postal`, `georeferencia` or `emissions_de_co2` to see if there are any documents pointed out that cannot be parsed.
- Try out the larger datasets to see how much data your hardware can quickly search.

## Further reading / references
- https://www.elastic.co/blog/kickstart-search-with-the-elastic-app-search-reference-ui
- https://www.elastic.co/blog/how-to-ingest-custom-data-into-elastic-workplace-search-a-simple-csv-example
- https://discuss.elastic.co/t/dec-5-2018-en-elastic-app-search-quick-and-easy-typeahead-search-with-elastic-app-search/159221
- https://www.elastic.co/es/webinars/best-practices-for-building-search-experiences-with-elastic-enterprise-search
- https://www.elastic.co/webinars/building-great-search-experiences-with-search-ui
- https://www.elastic.co/es/elasticon/archive/2020/global/building-great-search-experiences
