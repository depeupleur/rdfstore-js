#rdfstore-js


rdfstore-js is a pure Javascript implementation of a RDF graph store with support for the SPARQL query and data manipulation language.

    var rdfstore = require('rdfstore');
    
    rdfstore.create(function(store) {
      store.execute('LOAD <http://dbpedia.org/resource/Tim_Berners-Lee> INTO GRAPH <http://example.org/people>', function() {

        store.setPrefix('dbp', 'http://dbpedia.org/resource/');
        
        store.node(store.rdf.resolve('dbp:Tim_Berners-Lee'), "http://example.org/people", function(success, graph) {

          var peopleGraph = graph.filter(store.rdf.filters.type(store.rdf.resolve("foaf:Person")));
          
          store.execute('PREFIX rdf:  <http://www.w3.org/1999/02/22-rdf-syntax-ns#>\
                         PREFIX foaf: <http://xmlns.com/foaf/0.1/>\
                         PREFIX : <http://example.org/>\
                         SELECT ?s FROM NAMED :people { GRAPH ?g { ?s rdf:type foaf:Person } }',
                         function(success, results) {

                           console.log(peopleGraph.toArray()[0].subject.valueOf() === results[0].s.value);

                         });

        });

      });
    })

rdfstore-js can be executed in a web browser or can be included as a library in a node.js application.

The current implementation is far from complete but it already passes all the test cases for the SPARQL 1.0 query language and supports data manipulation operations from the SPARQL 1.1/Update version of the language.

Some other features included in the library are the following:

- SPARQL 1.0 support
- SPARQL 1.1/Update support
- Partial SPARQL 1.1 query support
- JSON-LD experimental parser
- Turtle/N3 parser
- W3C RDF Interfaces API
- RDF graph events API
- Parallel execution where WebWorkers are available
- Persistent storage using HTML5 LocalStorage

## SPARQL support

rdfstore-js supports at the moment SPARQL 1.0 and most of SPARQL 1.1/Update.
Only small parts of SPARQL 1.1 query has been implemented.

This is a list of the different kind of queries currently implemented:
  
- SELECT queries
- UNION, OPTIONAL clauses
- NAMED GRAPH identifiers
- LIMIT, OFFSET
- ORDER BY clauses
- SPARQL 1.0 filters and builtin functions
- variable aliases
- variable aggregation: MAX, MIN, COUNT, AVG, SUM functions
- GROUP BY clauses
- DISTINCT query modifier
- CONSTRUCT queries
- ASK queries
- INSERT DATA queries
- DELETE DATA queries
- DELETE WHERE queries
- WITH/DELETE/INSERT/WHERE queries
- LOAD queries
- CREATE GRAPH clasues
- DROP DEFAULT/NAMED/ALL/GRAPH clauses
- CLEAR DEFAULT/NAMED/ALL/Graph clauses

##Installation

To use the library in a node.js application, there is available a [package](http://search.npmjs.org/#/rdfstore) that can be installed using the NPM package manager:

    $npm install rdfstore

It is also possible to use rdfstore-js in a web application being executed in a browser. There is [minimized version](https://raw.github.com/antoniogarrote/rdfstore-js/master/dist/browser/rdf_store_min.js) of the library in a single Javascript file that can be linked from a HTML document. There is also a [minimized and gunzipped version](https://raw.github.com/antoniogarrote/rdfstore-js/master/dist/browser/rdf_store_min.js) available. Both versions have been compiled using Google's Closure Javascript compiler.
The persistent versions can be found [here (min)](https://raw.github.com/antoniogarrote/rdfstore-js/master/dist/browser_persistent/rdf_store_min.js).


##Building

The library can be built for different environments using the included Ruby script and configuration file. The JSON 1.5 Ruby gem is required by the build script.

To build the library for node.js execute the following command from the root directory of the project:

    $./make.rb nodejs

To build the library for the browser configuration, execute the following command:

    $./make.rb browser

To build the library for the browser, including support for persistent
storage execute this command:

    $./make.rb browser_persistent

The output of each configuration will be created in the dist subdirectory at the root path of the project.


## Tests

To execute the whole test suite of the library, including the DAWG test cases for SPARQL 1.0 and the test cases for SPARQL 1.1 implemented at the moment, the build script can be used:

    $./make.rb tests

The tests depend on [nodeunit](http://search.npmjs.org/#/nodeunit). That node.js library must be installed in order to run the tests.

You can also run the tests on the minimized version of the library with the command:

    $./make.rb test_min

Additionally, there are some smoke tests for both browser versions that can be found ithe 'browsertests' directory.

## API

This is a small overview of the rdfstore-js API.

###Store creation

    //nodejs only
    var rdfstore = require('rdfstore');

    // in the browser the rdfstore object
    // is already defined

    // alt 1
    rdfstore.create(function(store) {
      // the new store is ready
    });

    // alt 2
    var store = rdfstore.create()

    // alt 3
    new rdfstore.Store(function(store) {
      // the new store is ready
    });

    // alt 4
    store = new rdfstore.Store();

###Persistent store creation

In order to use persistent storage in the browser, an option named 'persitent' must be passed with value 'true' in the options for the store. An additional flag 'overwrite' indicates if the data for this store can be used to drop old data or read previously stored data. Optionally, a name for the store can also be passed as an argument. This name can be used to manipulate several persistent stores in the same browser.

At the moment, webworkers cannot be used with the persistent version of the store.

    new rdfstore.Store({persistent:true, name:'myappstore', overwrite:true}, function(store){
      // Passing overwrite:true to the options will make the store to drop all previous data.
      // Several stores can be used, providing different names for the stores
    }

###Query execution

    // simple query execution
    store.execute("SELECT * { ?s ?p ?o }", function(success, results){
      if(success) {
        // process results        
        if(results[0].s.token === 'uri') {
          console.log(results[0].s.value);
        }       
      }
    });

    // execution with an explicit default and named graph

    var defaultGraph = [{'token':'uri', 'vaue': graph1}, {'token':'uri', 'value': graph2}, ...];
    var namedGraphs  = [{'token':'uri', 'vaue': graph3}, {'token':'uri', 'value': graph4}, ...];

    store.executionWithEnvironment("SELECT * { ?s ?p ?o }",defaultGraph,
      namedGraphs, function(success, results) {
      if(success) {
        // process results
      }
    });
    
###Construct queries RDF Interfaces API

    var query = "CONSTRUCT { <http://example.org/people/Alice> ?p ?o } \
                 WHERE { <http://example.org/people/Alice> ?p ?o  }";

    store.execute(query, function(success, graph){
      if(graph.some(store.rdf.filters.p(store.rdf.resolve('foaf:name)))) {
        nameTriples = graph.match(null, 
                                  store.rdf.createNamedNode(rdf.resolve('foaf:name')),
                                  null);

        nameTriples.forEach(function(triple) {
          console.log(triple.object.valueOf());
        });                                  
      }
    });


###Loading remote graphs

rdfstore-js will try to retrieve remote RDF resources across the network when a 'LOAD' SPARQL query is executed.
The node.js build of the library will use regular TCP sockets and perform proper content negotiation. It will also follow a limited number of redirections.
The browser build, will try to perform an AJAX request to retrieve the resource using the correct HTTP headers. Nevertheless, this implementation is subjected to the limitations of the Same Domain Policy implemented in current browsers that prevents cross domain requests. Redirections, even for the same domain, may also fail due to the browser removing the 'Accept' HTTP header of the original request.
rdfstore-js relies in on the jQuery Javascript library to peform cross-browser AJAX requests. This library must be linked in order to exeucte 'LOAD' requests in the browser.  

    store.execute('LOAD <http://dbpedialite.org/titles/Lisp_%28programming_language%29>\
                   INTO GRAPH <lisp>', function(success){
      if(success) {
        var query = 'PREFIX foaf:<http://xmlns.com/foaf/0.1/> SELECT ?o \
                     FROM NAMED <lisp> { GRAPH <lisp> { ?s foaf:page ?o} }';
        store.execute(query, function(success, results) {
          // process results
        });
      }
    })

###High level interface

The following interface is a convenience API to work with Javascript code instead of using SPARQL query strings. It is built on top of the RDF Interfaces W3C API.

    /* retrieving a whole graph as JS Interafce API graph object */

    store.graph(graphUri, function(graph){
      // process graph
    });


    /* Exporting a graph to N3 (this function is not part of W3C's API)*/
    store.graph(graphUri, function(graph){
      var serialized = graph.toNT();
    });

     
    /* retrieving a single node in the graph as a JS Interface API graph object */

    store.node(subjectUri, function(graph) {
      //process node
    });
     
    store.node(subjectUri, graphUri, function(graph) {
      //process node
    });


     
    /* inserting a JS Interface API graph object into the store */

    // inserted in the default graph
    store.insert(graph, function(success) {}) ;

    // inserted in graphUri
    store.insert(graph, graphUri, function(success) {}) ;



    /* deleting a JS Interface API graph object into the store */

    // deleted from the default graph
    store.delete(graph, function(success){});

    // deleted from graphUri
    store.delete(graph, graphUri, function(success){});



    /* clearing a graph */
    
    // clears the default graph
    store.clear(function(success){});

    // clears a named graph
    store.clear(graphUri, function(success){});



    /* Parsing and loading a graph */

    // loading local data
    store.load("text/turtle", turtleString, function(success, results) {});

    // loading remote data
    store.load('remote', remoteGraphUri, function(success, results) {});



    /* Registering a parser for a new media type */

    // The parser object must implement a 'parse' function
    // accepting the data to parse and a callback function.

    store.registerParser("application/rdf+xml", rdXmlParser);

###RDF Interface API

The store object includes a 'rdf' object implementing a RDF environment as described in the [RDF Interfaces 1.0](http://www.w3.org/TR/rdf-interfaces/) W3C's working draft.
This object can be used to access to the full RDF Interfaces 1.0 API.

    var graph = store.rdf.createGraph();
    graph.addAction(rdf.createAction(store.rdf.filters.p(store.rdf.resolve("foaf:name")),
                                     function(triple){ var name = triple.object.valueOf();
                                                       var name = name.slice(0,1).toUpperCase() 
                                                       + name.slice(1, name.length);
                                                       triple.object = store.rdf.createNamedNode(name);
                                                       return triple;}));

    store.rdf.setPrefix("ex", "http://example.org/people/");
    graph.add(store.rdf.createTriple( store.rdf.createNamedNode(store.rdf.resolve("ex:Alice")),
                                      store.rdf.createNamedNode(store.rdf.resolve("foaf:name")),
                                      store.rdf.createLiteral("alice") ));

    var triples = graph.match(null, store.rdf.createNamedNode(store.rdf.resolve("foaf:name")), null).toArray();

    console.log("worked? "+(triples[0].object.valueOf() === 'Alice'));


###JSON-LD Support

rdfstore-js implements parsers for Turtle and JSON-LD. The specification of JSON-LD is still an ongoing effort. You may expect to find some inconsistencies between this implementation and the actual specification.

            jsonld = {
              "@context": 
              {  
                 "rdf": "http://www.w3.org/1999/02/22-rdf-syntax-ns#",
                 "xsd": "http://www.w3.org/2001/XMLSchema#",
                 "name": "http://xmlns.com/foaf/0.1/name",
                 "age": "http://xmlns.com/foaf/0.1/age",
                 "homepage": "http://xmlns.com/foaf/0.1/homepage",
                 "ex": "http://example.org/people/",
                 "@type":
                 {
                    "xsd:integer": "age",
                    "xsd:anyURI": "homepage",
                 }
              },
              "@": "ex:john_smith",
              "name": "John Smith",
              "age": "41",
              "homepage": "http://example.org/home/"
            };    

    store.setPrefix("ex", "http://example.org/people/");

    store.load("application/json", jsonld, "ex:text", function(success, results) {
      store.node("ex:john_smith", "ex:test", function(success, graph) {
        // process graph here
      });
    });

###Events API

rdfstore-js implements an experimental events API that allows clients to observe changes in the RDF graph and receive notifications when parts of this graph changes.
The two main event functions are *subscribe* that makes possible to set up a callback function that will be invoked each time triples matching a certain pattern passed as an argument are added or removed, and the function *startObservingNode* that will be invoked with the modified version of the node each time triples are added or removed from the node.

    var cb = function(event, triples){ 
      // it will receive a notifications where a triple matching
      // the pattern s:http://example/boogk, p:*, o:*, g:*
      // is inserted or removed.
      if(event === 'added') {
        console.log(triples.length+" triples have been added");  
      } else if(event === 'deleted') {
        console.log(triples.length+" triples have been deleted");  
      } 
    }
     
    store.subscribe("http://example/book",null,null,null,cb);
     
     
    // .. do something;
     
    // stop receiving notifications
    store.unsubscribe(cb);

The main difference between both methods is that *subscribe* receives the triples that have changed meanwhile *startObservingNode* receives alway the whole node with its updated triples. *startObservingNode* receives the node as a RDF Interface graph object.

    var cb = function(node){ 
      // it will receive the updated version of the node each
      // time it is modified.
      // If the node does not exist, the graph received will
      // not contain triples.
      console.log("The node has now "+node.toArray().length+" nodes");
    }
     
    // if only tow arguments are passed, the default graph will be used.
    // A graph uri can be passed as an optional second argument.
    store.startObservingNode("http://example/book",cb);
     
     
    // .. do something;
     
    // stop receiving notifications
    store.stopObservingNode(cb);

In the same way, there are *startObservingQuery* and *stopObservingQuery* functions that makes possible to set up callbacks for whole SPARQL queries. 
The store will try to be smart and not perform unnecessary evaluations of these query after quad insertion/deletions. Nevertheless too broad queries must be used carefully with the events API.

###WebWorkers

RDFStore includes experimental support for [webworkers](http://www.w3.org/TR/workers/) since version 0.4.0 in both browser and node.js versions.
The store can be initialized in a new thread using the *connect* method of the *Store* object.

The library will try to create a worker and will return a *connection* object providing the same interface of the *store* object.
If the creation of the worker fails, because webworkers support is not enabled in the platform/browser, a regular *store* object will be returned instead. Since both objects implement the same interface, client code can be wrote without taking into consideration the actual implementation of the store interface.

    Store.connect("/js/rdfstore_min.js", {}, function(success,store) {
        if(success) {
          // store is a connection to the worker
          console.log(store.isWebWorkerConnection === true);
        } else {
          // connection was not possible. A store object has been returned instead
        }       
        store.execute('INSERT DATA {  <http://example/book3> <http://example.com/vocab#title> <http://test.com/example> }', function(result, msg){
            store.execute('SELECT * { ?s ?p ?o }', function(success,results) {
              console.log(results.length === 1);
            });
        });
    });

The *connect* function can receive three arguments, the URL/Path where the script of the store is located, so the webworkers layer can load it, a hash with the aguments for the *Store.create* function that will be used in the actual creation of the store object and a callback that will be invoked with a success notification and the store implementation. In the Node.js version, it is not required to provide the path to the store script, the location of the store module will be provided by default.

At the moment, the usability of this feature is limited to those browsers where the web workers framework is enabled. It has been tested with the current version of Chrome and Firefox Aurora 8.0a2. 
Support in the Node.js version is provided by the webworkers module. 

Web worker threads execute in the browser in a very restrictive environment due to security reasons. One of these limitations is the impossibility to make AJAX requests. SPARQL queries and the store functionality to load remote resources cannot be executed inside a worker connection. In these cases it is required to load the resource from the main browser thread and then insert the data in the store throught the connection. These restrictions are not present in the Node.js version.


##Reusable modules. 

rdfstore-js is built from a collection of general purpose modules. Some of these modules can be easily extracted from the library and used on their own.

This is a listing of the main modules that can be re-used:

- src/js-trees: in-memory and persisten tree data structures: binary trees, red-black trees, b-trees, etc.
- src/js-sparql-parser: a SPARQL and a Turtle parsers built using the [PEG.js](http://pegjs.majda.cz/) parsing expression grammars library.
- src/js-trees/src/utils: a continuation passing style inspired library for different code flow constructions
- src/js-communication/src/json_ld_parser: JSON-LD parser implementation.
- src/js-query-enginesrc/rdf_js_interface: Javascript Interface 1.0 API implementation.

##Contributing

rdfstore-js is still at the beginning of its development. If you take a look at the library and find a way to improve it, please ping us. We'll be very greatful for any bug report or pull-request.

## Authors

Antonio Garrote, email:antoniogarrote@gmail.com, twitter:@antoniogarrote.

## License

Licensed under the [GNU Lesser General Public License Version 3 (LGPLV3)](http://www.gnu.org/licenses/lgpl.html), copyright Antonio Garrote 2011.
